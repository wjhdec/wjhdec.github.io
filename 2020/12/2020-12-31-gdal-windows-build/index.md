# windows下编译GDAL记录


:sectnums:
:icons: font

[abstract]
== 背景

这应该是今年最后一个帖子了，由于准备在java中使用gdal，而现在各种预编译版本中一般没有java的支持，需要自己编译。上班时用的 Linxu系统，很好编译，基本配置好环境，一个make就可以了，但是回家为了玩儿会儿游戏用的 Windows 系统，编译就比较曲折了，这里记录一下。

NOTE: 如果不需要java等支持，可以直接使用Conda的方式安装

== 使用版本

* Visual Studio 2017
* sqlite3 3.34.0
* proj 6.3.2
* gdal 3.1.4

== 编译流程

sqlite -> proj -> gdal

== 安装 Visual Studio 2017

这里安装的 Visual Studio 2017 的 Community 版本，这个版本自己用足够了，只安装的 c++ 支持，网络安装，还好网比较快，大概半个小时安装完成，不再细说

== 编译 sqlite3

安装这里有点坑，看其他帖子，很多写的是直接下载dll那个，但是我没测试成功过，且提供的编译好的exe只有32位，而proj编译需要exe文件

折腾了小半天后决定自己安装，直接下载 https://github.com/sqlite/sqlite/tree/version-3.34.0[源码]

先要安装 Tcl
https://www.activestate.com/products/tcl/downloads/[ActiveTcl下载地址]

安装完成后编译

编译时参考的 https://www2.sqlite.org/src/tree?ci=14864f2b8470fe98[这里]

其实就是在vs控制台环境里面执行 `nmake /f Makefile.msc TOP=..\sqlite`， exe 等文件就会生成到本页面。这时proj和gdal基本就可以用这个文件夹了。

== 编译 proj

这里使用的是 https://cmake.org/[Cmake] 编译，可以使用 cmake 的ui进行简单操作

config报错后修改以下几项

* EXE_SQLITE3 : D:/Codes/sqlite/sqlite3.exe
* SQLITE3_INCLUDE_DIR : D:\Codes\sqlite
* SQLITE3_LIBRARY : D:/Codes/sqlite/sqlite3.lib

可选修改项：

* CMAKE_INSTALL_PREFIX : D:\Work\proj

修改后在页面执行 Generate -> Open Project 打开vs工程

vs 工程中可直接在 `生成 -> 批生成` 中选择 ALL_BUILD 和 INSTALL 的 Release|x64 项目，执行后会直接把工程输出到 CMAKE_INSTALL_PREFIX 设置的路径中。这里基本完成 proj 的编译

== 编译 GDAL

这里暂时编译最简版本，以后需要再扩展编译

直接 https://proj.org/download.html[下载gdal源码]，gdal直接支持nmake编译，会比sqlite和proj简单点，不过编译过程需要几个项目的参与，需要另外下载，我目前需要的有：

* https://ant.apache.org/[ANT]
* https://sourceforge.net/projects/swig/[SWIG]
* JAVA

下载后可修改gdal源码中的 nmake.opt 文件，或直接在参数中指定。然后执行下面语句：

[source, shell]
----
nmake -f makefile.vc MSVC_VER=1910 WIN64=1 GDAL_HOME=D:\Work\gdal_3.1.4 PROJ_INCLUDE=-ID:\Work\proj_6.3.2\include PROJ_LIBRARY=D:\Work\proj_6.3.2\lib\proj.lib SQLITE_INC=-ID:\Work\sqlite SQLITE_LIB=D:\Work\sqlite\sqlite3.lib BINDINGS=java install
----

NOTE: 最后加上 `install` 才会把编译好的输出到 `GRADLE_HOME`，这里他的示例都没写

编译完成后就可以在 `GRADLE_HOME` 中看到编译好的可执行文件了

另：编译后的好像没找到 `jni` 相关的dll，明年再看了。


*祝大家新年快乐！*
