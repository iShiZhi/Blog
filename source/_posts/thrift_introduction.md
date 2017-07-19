---
title: Thrift 入门篇
categories: 开源项目
tags: [Thrift,RPC]
date: 2017-07-01
---

### 简介
Apache Thrift 是由 Facebook 开发的远程服务调用框架，它采用接口描述语言（IDL）定义并创建服务，支持可扩展的跨语言服务开发，所包含的代码生成引擎可以在多种语言中，如 C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, Smalltalk 等创建高效的、无缝的服务，其传输数据采用二进制格式，相对 XML 和 JSON 体积更小，对于高并发、大数据量和多语言的环境更有优势。

### 安装
- 下载 [boost](http://www.boost.org/) 并安装：

```bash
./bootstrap.sh
sudo ./b2 threading=multi address-model=64 variant=release stage install
```
- 下载 [libevent](http://libevent.org/) 并安装：

```bash
./configure --prefix=/usr/local
make
sudo make install
```
在执行 make 的时候可能会碰到一个问题：
```bash
/Library/Developer/CommandLineTools/usr/bin/make  all-am
  CC       libevent_openssl_la-bufferevent_openssl.lo
bufferevent_openssl.c:66:10: fatal error: 'openssl/bio.h' file not found
#include <openssl/bio.h>
         ^
1 error generated.
make[1]: *** [libevent_openssl_la-bufferevent_openssl.lo] Error 1
make: *** [all] Error 2
```
我的解决方案是执行如下命令：
```bash
./configure LDFLAGS='-L/usr/local/opt/openssl/lib' CPPFLAGS='-I/usr/local/opt/openssl/include'
```

- 下载 Thrift 并编译

```bash
./configure --prefix=/usr/local/ --with-boost=/usr/local --with-libevent=/usr/local
make
sudo make install
```
  这边可能会碰到一个问题，就是 mac 中默认安装了 bison 2.3 版本，并配置了路径在 path 中。
  需要安装最新的版本 3.0.4, 并将 /usr/bin 中的 bison 删除，将bison 3.0.4 复制到 /usr/bin 中：
```bash
brew install bison
cd /usr/bin
sudo mv bison bison.2.3
sudo cp /usr/local/Cellar/bison/3.0.4/bin/bison bxison
```
  如果安装了 Xcode，那有可能是 Xcode 自带的 bison。
  将 Xcode 的 bison 改名后编译，编译完以后再把名字改回来。
  路径：/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/
  此外，如果不需要其它语言的话，可以用 "--without" 选项将其排除，例如 "--without-perl"，否则可能因为本地环境配置的问题导致编译失败。


### 一个简单的例子
#### 例子的文件结构
```bash
├── client.py
├── gen-py
│   ├── __init__.py
│   └── helloworld
│       ├── HelloWorld-remote
│       ├── HelloWorld.py
│       ├── __init__.py
│       ├── constants.py
│       └── ttypes.py
├── helloworld.thrift
└── server.py
```

#### 定义 helloword.thrift 文件
```
service HelloWorld {
  string ping(),
  string say(1:string msg)
}
```

#### 自动生成代码
```bash
thrift --gen py helloword.thrift
```

#### 生成 server 端的代码
```python
#!/usr/bin/env python

import socket
import sys

sys.path.append('./gen-py')
from helloworld import HelloWorld
from helloworld.ttypes import *
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol
from thrift.server import TServer

class HelloWorldHandler:
    def ping(self):
        return "pong"

    def say(self, msg):
        ret = "Received: " + msg
        print ret
        return ret
```

#### 加入启动服务的代码
```python
handler = HelloWorldHandler()
processor = HelloWorld.Processor(handler)
transport = TSocket.TServerSocket("localhost", 9090)
tfactory = TTransport.TBufferedTransportFactory()
pfactory = TBinaryProtocol.TBinaryProtocolFactory()
server = TServer.TSimpleServer(processor, transport, tfactory, pfactory)
print "Starting thrift server in python..."
server.serve()
print "done!"
```

#### 生成 client 端的代码
```python
#!/usr/bin/env python

import sys

sys.path.append('./gen-py')
from helloworld import HelloWorld
from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

try:
    transport = TSocket.TSocket('localhost', 9090)
    transport = TTransport.TBufferedTransport(transport)
    protocol = TBinaryProtocol.TBinaryProtocol(transport)
    client = HelloWorld.Client(protocol)
    transport.open()

    print "client - ping"
    print "server - " + client.ping()
    print "client - say"
    msg = client.say("Hello!")
    print "server - " + msg

    transport.close()
except Thrift.TException, ex:
    print "%s" % (ex.message)
```

#### 运行代码
运行 server 端代码和 client 端代码，可以在 server 端看到如下日志：
```bash
Starting thrift server in python...
Received: Hello!
```
在 client 端看到如下日志：
```bash
client - ping
server - pong
client - say
server - Received: Hello!
```
