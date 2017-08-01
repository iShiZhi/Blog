---
title: '使用 Redis 实现分布式锁'
categories: 网站开发
tags: [Redis]
date: 2017-7-22
---

#### SETNX 命令
参考 Redis 的[命令手册](http://redisdoc.com/string/setnx.html)，相关说明如下：
```
SETNX key value

将 key 的值设为 value ，当且仅当 key 不存在。
若给定的 key 已经存在，则 SETNX 不做任何动作。
SETNX 是『SET if Not eXists』(如果不存在，则 SET)的简写。

可用版本：
>= 1.0.0
时间复杂度：
O(1)
返回值：
设置成功，返回 1 。
设置失败，返回 0 。
```

#### 简单实现
一个简单的想法是，执行命令 SETNX lock_name (current_time + lock_time)：
- 若返回 1，则设置成功，获得这个锁；
- 若返回 0，则等待或者直接返回失败。

#### 多个进程获得同一个锁
如果按照上述的实现方式，如果没有很好地实现锁的释放机制，则很有可能导致死锁的情况，具体的例子如下：
- P1 获得 lock_name 对应的锁
- P1 断开连接，没有正常释放锁
- P2 和 P3 同时发现锁已经超时
- P2 执行 DEL lock_name 删除锁
- P2 执行 SETNX lock_name 获得锁
- P3 执行 DEL lock_name 将 P2 刚刚设置的键 lock_name 删除
- P3 执行 SETNX lock_name 获得锁

在这种情况下，由于 P3 在 P2 获得锁前检测到了锁超时，因此执行了 DEL 命令，将 P2 后来设置的超时删除。而 P2 和 P3 同时获得了锁。


#### GETSET 命令
参考 Redis 的[命令手册](http://redisdoc.com/string/getset.html)，相关说明如下：
```
GETSET key value
将给定 key 的值设为 value ，并返回 key 的旧值(old value)。
当 key 存在但不是字符串类型时，返回一个错误。

可用版本：
>= 1.0.0
时间复杂度：
O(1)
返回值：
返回给定 key 的旧值。
当 key 没有旧值时，也即是， key 不存在时，返回 nil 。
```

#### 解决方案
我们需要替换对锁超时的情况的处理，按时间先后顺序如下：
- P1 获得 lock_name 对应的锁
- P1 断开连接，没有正常释放锁
- P2 和 P3 同时发现锁已经超时
- P2 执行 GETSET lock_name (current_time + lock_time)
- P3 执行 GETSET lock_name (current_time + lock_time)
- P3 获取返回结果，lock_name 对应的旧值小于检测时间，说明设置成功，获得锁
- P2 获取返回结果，lock_name 对应的旧值是 P3 设置的结果，大于检测时间，说明已有其它线程获得锁，P2 获取失败

利用 GETSET 返回旧值的特性，可以在检测到超时的时候，执行 GETSET 设置新的超时时间，即使有多个进程同时执行 GETSET，最终也只会有一个判定设置成功，从而获得锁。


#### Python 实现
```python
class Lock(object):
    """
    A shared, distributed Lock. Using Redis for locking allows the Lock
    to be shared across processes and/or machines.

    It's left to the user to resolve deadlock issues and make sure
    multiple clients play nicely together.
    """

    LOCK_FOREVER = float(2 ** 31 + 1)  # 1 past max unix time

    def __init__(self, redis, name, timeout=None, sleep=0.1):
        """
        Create a new Lock instnace named ``name`` using the Redis client
        supplied by ``redis``.

        ``timeout`` indicates a maximum life for the lock.
        By default, it will remain locked until release() is called.

        ``sleep`` indicates the amount of time to sleep per loop iteration
        when the lock is in blocking mode and another client is currently
        holding the lock.

        Note: If using ``timeout``, you should make sure all the hosts
        that are running clients have their time synchronized with a network
        time service like ntp.
        """
        self.redis = redis
        self.name = name
        self.acquired_until = None
        self.timeout = timeout
        self.sleep = sleep
        if self.timeout and self.sleep > self.timeout:
            raise LockError("'sleep' must be less than 'timeout'")

    def __enter__(self):
        return self.acquire()

    def __exit__(self, exc_type, exc_value, traceback):
        self.release()

    def acquire(self, blocking=True):
        """
        Use Redis to hold a shared, distributed lock named ``name``.
        Returns True once the lock is acquired.

        If ``blocking`` is False, always return immediately. If the lock
        was acquired, return True, otherwise return False.
        """
        sleep = self.sleep
        timeout = self.timeout
        while 1:
            unixtime = mod_time.time()
            if timeout:
                timeout_at = unixtime + timeout
            else:
                timeout_at = Lock.LOCK_FOREVER
            timeout_at = float(timeout_at)
            if self.redis.setnx(self.name, timeout_at):
                self.acquired_until = timeout_at
                return True
            # We want blocking, but didn't acquire the lock
            # check to see if the current lock is expired
            existing = float(self.redis.get(self.name) or 1)
            if existing < unixtime:
                # the previous lock is expired, attempt to overwrite it
                existing = float(self.redis.getset(self.name, timeout_at) or 1)
                if existing < unixtime:
                    # we successfully acquired the lock
                    self.acquired_until = timeout_at
                    return True
            if not blocking:
                return False
            mod_time.sleep(sleep)

    def release(self):
        "Releases the already acquired lock"
        if self.acquired_until is None:
            raise ValueError("Cannot release an unlocked lock")
        existing = float(self.redis.get(self.name) or 1)
        # if the lock time is in the future, delete the lock
        delete_lock = existing >= self.acquired_until
        self.acquired_until = None
        if delete_lock:
            self.redis.delete(self.name)
```

