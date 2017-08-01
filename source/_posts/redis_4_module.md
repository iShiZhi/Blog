---
title: 'Redis 4.0 新功能：模块系统'
categories: 开源项目
tags: [Redis]
date: 2017-07-29
---

### 引入模块的动机
Redis 4.0 中出现的最大的变化，也是促使 antirez 不使用 3.4，而是直接使用 4.0 来标记这个版本的原因，就是引入模块系统。
其实早在 Redis 1.0 发布的时候，antirez 就在未来可能引入的功能里提到了“可载入的模块”，但是在后续的版本中一直没有实现。他自述的理由如下：
- 不同版本的 API 的兼容性问题
- 低质量的模块可能造成的系统冲突
- 缺乏可拓展的模块标识系统

因此，从 Redis 1.0 发布以后，antirez 一直都在使用 LUA 脚本来实现功能。
但正是由于这些实践，让他发现使用脚本开发只能组合已有的功能，并不能有效地拓展当前框架下没有的功能，于是下定决心写一个模块系统。
设计这个模块系统最难的，也是工作量最大的点在于：需要对于 Redis 内核进行封装，使得 API 独立于 Redis 内核，达到 Redis 内核升级以后也不会影响已经开发好的模块。

### 模块加载
可以在 redis.conf 中配置载入模块：
```bash
loadmodule <path> [args...]
```

或者在运行时动态载入模块：
```bash
MODULE LOAD <path> [args...]
```

可以来看看 MODULE LOAD 命令的内部实现 (module.c)，主要分为如下几个步骤：
- 加载指定路径的动态库
- 检查并回调模块实现的 RedisModule_OnLoad 函数
- 注册模块到模块列表，方便后续卸载

```c
int moduleLoad(const char *path, void **module_argv, int module_argc) {
    int (*onload)(void *, void **, int);
    void *handle;
    RedisModuleCtx ctx = REDISMODULE_CTX_INIT;

    // 加载指定路径的动态库
    handle = dlopen(path,RTLD_NOW|RTLD_LOCAL);
    if (handle == NULL) {
        serverLog(LL_WARNING, "Module %s failed to load: %s", path, dlerror());
        return C_ERR;
    }
    // 检查 RedisModule_OnLoad 函数是否实现
    onload = (int (*)(void *, void **, int))(unsigned long) dlsym(handle,"RedisModule_OnLoad");
    if (onload == NULL) {
        serverLog(LL_WARNING,
            "Module %s does not export RedisModule_OnLoad() "
            "symbol. Module not loaded.",path);
        return C_ERR;
    }
    // 回调模块的 RedisModule_OnLoad 函数
    if (onload((void*)&ctx,module_argv,module_argc) == REDISMODULE_ERR) {
        if (ctx.module) moduleFreeModuleStructure(ctx.module);
        dlclose(handle);
        serverLog(LL_WARNING,
            "Module %s initialization failed. Module not loaded",path);
        return C_ERR;
    }

    // 注册模块到模块列表，方便后续卸载
    dictAdd(modules,ctx.module->name,ctx.module);
    ctx.module->handle = handle;
    serverLog(LL_NOTICE,"Module '%s' loaded from %s",ctx.module->name,path);
    moduleFreeContext(&ctx);
    return C_OK;
}
```

### 编写模块
模块必须实现 RedisModule_OnLoad，以便在 Redis 调用模块的时候正确处理模块的初始化。以下是一个简单地模块：
```
#include "redismodule.h"
#include <stdlib.h>

int HelloworldRand_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    RedisModule_ReplyWithLongLong(ctx,rand());
    return REDISMODULE_OK;
}

int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    if (RedisModule_Init(ctx,"helloworld",1,REDISMODULE_APIVER_1)
        == REDISMODULE_ERR) return REDISMODULE_ERR;

    if (RedisModule_CreateCommand(ctx,"helloworld.rand",
        HelloworldRand_RedisCommand) == REDISMODULE_ERR)
        return REDISMODULE_ERR;

    return REDISMODULE_OK;
}
```

在 RedisModule_OnLoad 中主要进行了两个任务：
- 模块初始化：调用 RedisModule_Init 函数，将 Redis 对外提供的接口暴露给模块使用
- 添加命令到 Redis，整合模块提供的功能到 Redis 的命令处理过程中
这是 Redis 内部实现创建 Command 的函数，它的主要逻辑是：创建一个 RedisModuleCommandProxy 并把它的 rediscmd 作为命令的处理函数。
而在 RedisModuleCommandDispatcher 中，会先创建上下文，调用具体的实现函数，然后调用 callback，销毁上下文。

```c
int RM_CreateCommand(RedisModuleCtx *ctx, const char *name, RedisModuleCmdFunc cmdfunc, const char *strflags, int firstkey, int lastkey, int keystep) {
    struct redisCommand *rediscmd;
    RedisModuleCommandProxy *cp;
    ...
    // 忽略次要的代码路径

    cp = zmalloc(sizeof(*cp));
    cp->module = ctx->module;
    // 真正的处理函数
    cp->func = cmdfunc;
    cp->rediscmd = zmalloc(sizeof(*rediscmd));
    cp->rediscmd->name = cmdname;
    // 命令处理函数设置为 RedisModuleCommandDispatcher
    cp->rediscmd->proc = RedisModuleCommandDispatcher;
    cp->rediscmd->arity = -1;
    cp->rediscmd->flags = flags | CMD_MODULE;
    cp->rediscmd->getkeys_proc = (redisGetKeysProc*)(unsigned long)cp;
    cp->rediscmd->firstkey = firstkey;
    cp->rediscmd->lastkey = lastkey;
    cp->rediscmd->keystep = keystep;
    cp->rediscmd->microseconds = 0;
    cp->rediscmd->calls = 0;
    dictAdd(server.commands,sdsdup(cmdname),cp->rediscmd);
    dictAdd(server.orig_commands,sdsdup(cmdname),cp->rediscmd);
    return REDISMODULE_OK;
}

void RedisModuleCommandDispatcher(client *c) {
    RedisModuleCommandProxy *cp = (void*)(unsigned long)c->cmd->getkeys_proc;
    RedisModuleCtx ctx = REDISMODULE_CTX_INIT;

    ctx.module = cp->module;
    ctx.client = c;
    // 调用真正的处理函数
    cp->func(&ctx,(void**)c->argv,c->argc);
    moduleHandlePropagationAfterCommandCallback(&ctx);
    moduleFreeContext(&ctx);
}
```

值得一提的是，在模块中调用的函数是 RedisModule_CreateCommand，而上述的函数是 RM_CreateCommand。这是因为 Redis 做了一层重命名，内部实现是 RM_ 开头，而对外暴露的时候使用 RedisModule_ 开头，具体的转化在 RedisModule_Init 里完成。

### 自定义数据结构
创建一个新的数据结构时，需要定义好几个回调函数。Redis 并不关心外部的数据是如何操作的，只需要关心数据要怎么来进行持久化以及释放，具体的操作由命令决定。例子如下：
```c
int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    REDISMODULE_NOT_USED(argv);
    REDISMODULE_NOT_USED(argc);

    if (RedisModule_Init(ctx,"hellotype",1,REDISMODULE_APIVER_1)
        == REDISMODULE_ERR) return REDISMODULE_ERR;

    RedisModuleTypeMethods tm = {
        .version = REDISMODULE_TYPE_METHOD_VERSION,
        .rdb_load = HelloTypeRdbLoad,
        .rdb_save = HelloTypeRdbSave,
        .aof_rewrite = HelloTypeAofRewrite,
        .free = HelloTypeFree
    };
    HelloType = RedisModule_CreateDataType(ctx,"hellotype",0,&tm);
    if (HelloType == NULL) return REDISMODULE_ERR
}
```

### 已有模块
目前已经有人使用这个功能开发了各种各样的模块， 比如 Redis Labs 开发的一些模块就可以在 http://redismodules.com 看到， 此外 antirez 自己也使用这个功能开发了一个神经网络模块： https://github.com/antirez/neural-redis

### 参考资料
- [Redis Loadable Modules System](http://antirez.com/news/106)
- [Redis Modules: an introduction to the API](https://redis.io/topics/modules-intro)
- [Modules API reference](https://redis.io/topics/modules-api-ref)
