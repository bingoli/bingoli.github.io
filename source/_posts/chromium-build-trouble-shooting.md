---
title: 编译Chromium的问题处理
date: 2019-11-18 22:35:56
tags:
 - Chromium
 - 编译
 - gn
 - ninja
---
## 前言
Chromium编译时，会遇到各种各样的问题，本文用例记录在实践过程中碰到的问题，以便后面碰到类似的问题，能够更快速解决。
## 编译问题
### windows下的python组件
> Traceback (most recent call last):
>   File "../../build/toolchain/win/tool_wrapper.py", line 51, in <module>
>     import win32file    # pylint: disable=import-error
> ImportError: No module named 

解决方法：安装win32file模块。
> python -m pip install 

### win10 sdk安装失败
> C:/Python27/python.exe ../../build/win/copy_cdb_to_output.py cdb x64
> Traceback (most recent call last):
>   File "../../build/win/copy_cdb_to_output.py", line 121, in <module>
>     sys.exit(main())
>   File "../../build/win/copy_cdb_to_output.py", line 117, in main
>     return _CopyCDBToOutput(sys.argv[1], sys.argv[2])
>   File "../../build/win/copy_cdb_to_output.py", line 96, in _CopyCDBToOutput
>     _CopyImpl('cdb.exe', output_dir, src_dir)
>   File "../../build/win/copy_cdb_to_output.py", line 46, in _CopyImpl
>     shutil.copy(source, target)
>   File "C:\Python27\lib\shutil.py", line 119, in copy
>     copyfile(src, dst)
>   File "C:\Python27\lib\shutil.py", line 82, in copyfile
>     with open(src, 'rb') as fsrc:
> IOError: [Errno 2] No such file or directory: 'C:\\Program Files (x86)\\Windows Kits\\10\\Debuggers\\x64\\cdb.exe'      [184/2536] CC obj/third_party/boringssl/boringssl/bcm.obj
> ninja: build stopped: subcommand failed.

问题原因：未安装win10sdk，或者未安装x64的sdk。
解决方法：重新安装win10 sdk，下载地址：
https://developer.microsoft.com/zh-cn/windows/downloads/windows-10-sdk 

### gn的问题
> gn.py: Could not find checkout in any parent of the current path.
> This must be run inside a checkout.
问题原因：gn需要去查找buildtools的路径，首先会查找环境变量CHROMIUM_BUILDTOOLS_PATH的值，然后查找.gclient所在目录，并查找该目录下的src/buildtools，如果.gclient不存在，就会出现上面的错误。在把chromium的代码拷贝到github的时候，有时会不注意这个细节。
解决方案：

有2种解决方法
* 设置环境变量CHROMIUM_BUILDTOOLS_PATH的值未src目录下buildtools的路径。
* 在src的上一级目录，保留.gclient和.gclient_entries。

## 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli