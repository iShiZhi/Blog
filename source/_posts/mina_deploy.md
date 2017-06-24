---
title: Mina自动化部署
categories: 运维相关
tags: [Ruby on Rails]
date: 2016-09-03
---

最近给公司的架构做了下调整，把之前堆在一台服务器上的后台任务、几个产品线做了下拆分，加了几台机器。于是原先使用 Bash 脚本进行部署的方式就显得有些麻烦，就用 Mina 折腾了一下自动化部署，顺便在这里把 Mina 的一些用法记录一下。它的原理很简单，就是生成一个 Bash 脚本文件，然后 SSH 到服务器上运行这个脚本文件。

## 安装

```bash
$ gem install mina
$ mina
```

## 配置部署文件

### 创建 config/deploy.rb

```bash
$ mina init
Created config/deploy.rb.
```

### 在服务器上创建文件夹

```bash
$ ssh username@your.server.com
# Once in your server, create the deploy folder:
~@your.server.com$ mkdir /var/www/flipstack.com
~@your.server.com$ chown -R username /var/www/flipstack.com
```

### 运行 mina setup

```bash
$ mina setup
-----> Creating folders... done.
```

### 进行部署

```bash
$ mina deploy
-----> Deploying to 2012-06-12-040248
       ...
       Lots of things happening...
       ...
-----> Done.
```

## 部署过程

```bash
$ mina deploy --verbose
-----> Creating the build path
       $ mkdir tmp/build-128293482394
-----> Cloning the Git repository
       $ git clone https://github.com/nadarei/flipstack.git . -n --recursive
       Cloning... done.
-----> Installing gem dependencies using Bundler
       $ bundle install --without development:test
       Using i18n (0.6.0)
       Using multi_json (1.0.4)
       ...
       Your bundle is complete! It was installed to ./vendor/bundle
-----> Moving to releases/4
       $ mv "./tmp/build-128293482394" "releases/4"
-----> Symlinking to current
       $ ln -nfs releases/4 current
-----> Launching
       $ cd releases/4
       $ sudo service nginx restart
-----> Done. Deployed v4
```

从上面的日志里，我们可以看到 Mina 的部署过程分为如下几个步骤：

- 创建一个临时文件夹，在这个临时文件夹里面完成代码的拷贝以及各类的准备工作；
- 将临时文件夹移动到 releases，并按照版本号进行重新命名，链接到 current；
- 运行部署脚本，完成部署。

## deploy.rb 文件

注意到，deploy.rb实际上是一个 Rake File，因此可以使用 Ruby 的语法进行编写，主要使用的命令如下：

- set [variable_name] [value] 对使用的变量进行赋值

```ruby
set :domain, 'flipstack.com'
queue 执行一条 Bash 脚本命令
```

- task 创建一个任务，这个任务可以在其它任务中通过 invoke 调用

```ruby
task :down do
  invoke :maintenance_on
  invoke :restart
end
task :maintenance_on
  queue 'touch maintenance.txt'
end
task :restart
  queue 'sudo service restart apache'
end
```

在上面一个部分介绍了 Mina 部署的过程，因此，在 task 里，同样对应了不同阶段：
```ruby
# 定义 SSH 的参数，以及 GIT 的参数
set :domain, 'flipstack.com'
set :user, 'flipstack'
set :deploy_to, '/var/www/flipstack.com'
set :repository, 'http://github.com/flipstack/flipstack.git'
task :deploy do
  deploy do
    # 放置在这里的命令是在创建临时文件夹以后，移动到 releases 文件夹以前执行的
    # 因此，注意这里执行的命令对应的路径
    invoke :'git:clone'
    invoke :'bundle:install'
    # 这里是移动到 releases 文件夹，并做完到 current 的链接以后执行的部署命令
    to :launch do
      queue 'touch tmp/restart.txt'
    end
    # 这里是部署失败以后做的清理工作
    to :clean do
      queue 'log "failed deployment"'
    end
  end
end
```

## 预定义的一些任务

Mina 已经将一些常见的任务进行了定义，我们只要通过 invoke 进行调用即可，常用的任务如下：

- git:clone
- bundle:install
- rails:db_migrate
- rails:assets_precompile
- whenever:update

Mina 对这些常用的任务进行了不同程度的优化，例如执行git:clone的时候，如果服务器上已经有 repository 了，则只会进行git fetch，而不会真的执行 clone 这个极其耗时的操作；另外，对于 assets，Mina 会进行检查，看看上一次编译以后是否有更新，如果没有更新的话，则会自动跳过这一次的 compile。

另外，还有一些 Gem 也可以使用：

- [Delayed Job](https://github.com/d4be4st/mina-delayed_job)
- [RPush](https://github.com/d4rky-pl/mina-rpush)

