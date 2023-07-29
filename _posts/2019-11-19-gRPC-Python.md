---
layout:     post
title:      "如何使用Python访问gRPC服务"
subtitle:   "How to use gRPC in Python"
date:       2019-11-19
author:     LiuShuo
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Python
---
    
> gRPC是一种比较流行的RPC通信框架，由谷歌公司开源，它提供了对Java、C++以及Python等常用语言的支持。本文主要梳理在Python环境下如何使用gRPC进行通信。

### 相关工具安装
#### Python3
如果使用Mac，请使用brew安装：
> brew install python

#### Pip
pip 是The Python Packaging Authority (PyPA) 推荐的 Python 包管理工具。

从 Python 2 版本 >=2.7.9 或 Python 3 版本 >=3.4 开始，官网的安装包中已经自带了 pip，在安装时用户可以直接选择安装。或者如果使用由 virtualenv 或者 pyvenv 创建的 Virtual Environment，那么 pip 也是被默认安装的。

如果没有在安装的时候，选择上安装pip，那么也可以从本地安装。例如，直接使用get-pip.py进行安装。首先从官网下载get-pip.py，然后直接运行python get-pip.py即可。

可以直接在本机执行如下命令查看pip的版本：
>  pip --version
>
>  pip 18.1 from /usr/local/lib/python3.7/site-packages/pip (python 3.7)

常用命令：
```
安装
pip install SomePackage              # 最新版本
pip install SomePackage==1.0.4       # 指定版本
pip install 'SomePackage>=1.0.4'     # 最小版本

卸载安装包命令
pip uninstall <包名> 或 pip uninstall -r requirements.txt

升级
pip install -U <包名> 或：pip install <包名> --upgrade

查看
pip freeze #查看已经安装的包及版本信息

列出可以升级的包
pip list -o

搜索包
pip search <pkgName>

查看包详情
pip show <pkgName>
```
另外关于pip源多说一句，由于国外官方pypi经常被墙，导致不可用，所以我们最好是将自己使用的pip源更换一下，这样就能解决被墙导致的装不上库的烦恼。
> 例如：pip install -i https://pypi.tuna.tsinghua.edu.cn/simple gevent，这样就会从清华这边的镜像去安装gevent库。
  
另外还可以直接使用pip.conf永久配置pip源：
```
linux下，修改 ~/.pip/pip.conf (没有就创建一个)， 修改 index-url，内容如下：

 [global]
 index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```
非virtualenv环境下，pip.conf的用户配置在 ${HOME}/.config/pip/pip.conf或${HOME}/.pip/pip.conf

在Unix系统里，全局配置在/etc/pip.conf，更多参考官方[文档](https://pip.pypa.io/en/stable/user_guide/#config-file)。

#### Virtualenv
在开发Python应用程序的时候，系统安装的Python3只有一个版本：3.4。所有第三方的包都会被pip安装到Python3的site-packages目录下。

如果我们要同时开发多个应用程序，那这些应用程序都会共用一个Python，就是安装在系统的Python 3。如果应用A需要jinja 2.7，而应用B需要jinja 2.6怎么办？

这种情况下，每个应用可能需要各自拥有一套“独立”的Python运行环境。virtualenv就是用来为一个应用创建一套“隔离”的Python运行环境。

更多资料可以参考这篇[文章](https://www.liaoxuefeng.com/wiki/1016959663602400/1019273143120480)。

简单安装一行命令
> pip3 install virtualenv

常用命令
```
> 创建一个虚拟环境，并放到本地venv文件夹内
> virtualenv venv venv 
>
> 激活venv环境，此时所有命令如python、pip等都使用venv环境内的命令而不是系统命令
> source venv/bin/activate 
>
> 禁用venv环境，此时python的命令将使用系统默认
> deactivate 
```
另外可以使用python命令调用venv来完成同样的功能
> python3 -m venv myvenv

#### gRPC
执行如下命令安装 `grpcio` ，它会自动安装 `protobuf` 包：
> pip install grpcio

生成proto文件对应的Python文件执行如下命令（使用了protobuf的编译器）：
> python -m grpc_tools.protoc -I../ --python_out=. --grpc_python_out=. ./grpchello.proto

结果将保存到当前目录下，包括grpchello_pb2.py和grpchello_pb2_grpc.py两个文件。
#### 中文乱码
即默认情况下，打印protobuf对象时，对于非asc码，会以转义符方式打印，[issue](https://github.com/google/protobuf/issues/2277)
> I'm pretty sure that's working as intended: by default we print all non-ascii characters in escaped sequence. 
  
只需要添加两行：
> from google.protobuf import text_format
>
> print(text_format.MessageToString(msg, as_utf8 = True))

这样msg结构里的文本属性会自动解析为正确的汉字。

如果需要单独获得desc内容进行其他操作则只能用一种过渡的方式，即定义一个单独的PB结构用于存放这个string类型属性，
然后将desc的内容赋值给它，然后再text_format.MessageToString(newPbMsg, as_utf8=True)，e.g.

```python
# 初始化一个PB结构TestMessage，内部属性只有string类型的属性，即text字段
# 这里将原来的PB对象的字符串属性desc赋值给TextMessage的text字段，然后实例化TextMessage对象
text_msg = im_ks_message_notice_pb2.TestMessage(text=somePbObj.desc) 
# 转化pb为文本，用utf8编码
text = text_format.MessageToString(text_msg, as_utf8=True)
print(text)
```

### References
- pip用户指南: https://pip.pypa.io/en/stable/user_guide/
- python wheel使用指南: https://wheel.readthedocs.io/en/stable/
- tox: https://tox.readthedocs.io/en/latest/
- pytest：https://docs.pytest.org/en/latest/
- 多版本py: https://github.com/pyenv/pyenv
- 生成文档：https://github.com/sphinx-doc/sphinx/
- 代码风格：https://www.python.org/dev/peps/pep-0008/
- 代码检查：https://www.pylint.org/

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
