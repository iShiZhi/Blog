---
title: Python 共享变量
categories: 开发语言
tags: [Python]
date: 2017-07-16
---

今天开发的时候碰到一个坑，在一个数据库模块中定义了一个 Redis 的连接，初始化以后，在具体的业务模块中获取 Redis 连接怎么都是 None。经过排查，发现自己对于 Python 的多模块共享变量的 import 机制缺乏了解，于是查阅资料，做个总结。

### 碰到的问题
简化以后的伪代码如下：

```python
# redis.py

redis_conn = None

def config_redis_conn(conf):
    redis_conn = init_new_redis(conf)
```

```python
# bootstrap.py

from redis import config_redis_conn

if __name__ == '__main__':
    config_redis_conn({})
```

```python
# business_logic.py
from redis import redis_conn

def logic():
    redis_conn.set('A', 'B')
```

在 business_logic 使用 redis_conn 的时候，会发现 redis_conn 是 None，即使已经经过 bootstrap 的初始化。

### Python 的 import 机制
Python import 包的机制是：import 进来的模块和默认的系统模块，都放在 sys.module 这个字典里面。当模块再次 import 的时候，会先去 sys.module 里面检查是否已经 import 了，如果已经 import 了，就不再重复 import，否则就 import 进来。
from redis import redis_conn 这样的方式引用，就相当于在 business_logic.py 文件中写了一行代码 redis_conn = None，此时 redis_conn 就是 business_logic.py 自己命名空间中的变量。所以 redis_conn 只在 business_logic.py 中有效，因此即使在 bootstrap.py 中初始化了 redis_conn，business_logic.py 中的 redis_conn 依旧没有改变。换种说法，实际上
```python
from redis import redis_conn
```
等同于
```python
improt redis
redis_conn = redis.redis_conn
```

所以，如果需要共享变量，就不要使用 from file import x 这种形式，而是使用 import file，然后就可以通过 file.x 来使用，然后 file.x='abc' 进行修改。这样都这样处理全局性的变量就可以共享的。也就是保持一个独立的namespace，这样python不会再次导入，从而实现共享。
