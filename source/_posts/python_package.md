---
title: Python 包管理
categories: 开发语言
tags: [Python]
date: 2017-06-28
---

之前在用 Ruby，Java 和 NodeJs 的时候，都有比较明确的包或模块管理的概念。换工作以后开始使用 Python，由于我们的开发机是一个多人使用的环境（物理机），因此经常会发生包路径的优先级被修改的情况。而我之前对于 Python 的包管理并不熟悉，因此当我在使用环境的时候，被人修改了 pythonpath，结果让我排查了很久才找到原因。因此，特地查阅一下资料，发现有一篇[Python开发生态环境简介](https://github.com/dccrazyboy/pyeco/blob/master/pyeco.rst)写得很详细，在这里留个记录。

### 理解包

Python 没有一个默认的包管理设施。事实上，包的概念在 Python 中是相当弱的。Python 代码被组织为模块。一个模块可能由包含一个函数的单一文件组成，也可能由包含多个模块的目录组成。 包和模块的区别非常小，并且每个模块都能被理解为包。

那么包和模块的区别到底是什么（如果有的话）？为了明白这个，你首先应该明白 Python 是如何查找模块的。

如同别的编程环境一样，Python中的一些函数和类（例如str,len,Exception等）在全局（叫做内置函数）都是可用的。 别的就需要通过手动 import 进来。例如:

```python
>>> import os
>>> from os.path import basename, dirname
```

这个包一定存在你的机子上，这样才能被 import 语句导入。但Python是如何知道这些模块的位置呢？这些位置信息在你安装Python虚拟机时就被自动设置好了，并且依赖于你的目标平台。包的路径可以在 sys.path 中查询。下面是在我的笔记本上的结果，运行环境是 Ubuntu 11.10。

```python
>>> import sys
>>> print sys.path
['',
 '/usr/lib/python2.7',
 '/usr/lib/python2.7/plat-linux2',
 '/usr/lib/python2.7/lib-tk',
 '/usr/lib/python2.7/lib-old',
 '/usr/lib/python2.7/lib-dynload',
 '/usr/local/lib/python2.7/dist-packages',
 '/usr/lib/python2.7/dist-packages',
 '/usr/lib/python2.7/dist-packages/PIL',
 '/usr/lib/python2.7/dist-packages/gst-0.10',
 '/usr/lib/python2.7/dist-packages/gtk-2.0',
 '/usr/lib/pymodules/python2.7',
 '/usr/lib/python2.7/dist-packages/ubuntu-sso-client',
 '/usr/lib/python2.7/dist-packages/ubuntuone-client',
 '/usr/lib/python2.7/dist-packages/ubuntuone-control-panel',
 '/usr/lib/python2.7/dist-packages/ubuntuone-couch',
 '/usr/lib/python2.7/dist-packages/ubuntuone-installer',
 '/usr/lib/python2.7/dist-packages/ubuntuone-storage-protocol']
 ```
这里给出了 Python 搜索包的路径。它将从最上面开始找，直到找到一个名字相符的。 这表明如果两个不同的路径分别包含了两个具有相同名字的包，搜索将在找到第一个名字的时候停止，然后将永远不会往下查找。

正如你所猜的，包搜索路径很容易被劫持，为了确保 Python 首先载入你的包，所需做的如下：
```python
>>> sys.path.insert(0, '/path/to/my/packages')
```
尽管这个方法在很多情况下都很好用，但一定要小心不要滥用。 只有当必要时再使用！不要滥用！
site 模块控制包的搜索路径。当 Python 虚拟机初始化时它会自动被导入。

### PYTHONPATH 变量

PYTHONPATH 是一个用来增加默认包搜索目录的环境变量。可以认为它是对于 Python 的一个特殊的 PATH 变量。 它仅仅是一个通过“:”分割，包含 Python 模块目录的列表（并不是类似于 sys.path 的 Python list）。 它可能就类似下面这样：
```bash
export PYTHONPATH=/path/to/some/directory:/path/to/another/directory:/path/to/yet/another/directory
```
有时候你可能并不想覆盖掉现存的 PYTHONPATH ，而仅仅是希望添加新目录到头部或尾部。
```bash
export PYTHONPATH=$PYTHONPATH:/path/to/some/directory    # Append
export PYTHONPATH=/path/to/some/directory:$PYTHONPATH    # Prepend
```
PYTHONPATH，sys.path.insert 这些方法并非完美，我们最好也不要用这些方法。使用它们，你可能可以解决本地的开发环境问题，但它在别的环境下也许并不适用。

我们现在明白的Python如何找到安装的包路径，现在让我们回到开始那个问题。 模块和包的区别到底是什么？包是一个模块或模块/子模块的集合，一般情况下被压缩到一个压缩包中。 其中包含：
1. 依赖信息
2. 将文件拷贝到标准的包搜索路径的指令
3. 编译指令(如果在安装前代码必须被编译的话


