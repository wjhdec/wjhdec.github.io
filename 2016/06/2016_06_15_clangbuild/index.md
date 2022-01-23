# Clang 编译


最近准备用C改一些东西，准备尝试一下LLVM和Clang，正好写一下它的编译。

<!-- more -->

## 1. 编译环境

　 　由于在个人电脑上编译，为了不影响我那几个RPG和时常挂着却一直没动静的QQ，还是暂定用Windows系统编译。由于个人习惯（主要是懒），实在不想装 Visual Studio 那样的巨无霸，干脆直接用MinGW，直接复制粘贴。资源不太好找，墙外下了好几次才下载下来。我放到了我的资源里面，地址：[MinGW-W64下载](https://download.csdn.net/download/happywjh666/9520346)，后加：最近CSDN有点坑，原来我设置的是免积分的，现在自动设成了3个，大家可以用[tdm-gcc](http://tdm-gcc.tdragon.net/download)

　 　把我现在的编译环境整理一下：

* 系统：Windows 7 64位
* 编译器：MinGW-W64-builds-4.2.0
* CMake：CMake 3.5.2 [下载地址](https://cmake.org/download/)

　 　MinGW 下载下来后需在环境变量的PATH里加一下。CMake如果是ZIP的话最好也加到PATH里，方便使用。配置方法不难，不再详谈。

## 2. LLVM源码下载

llvm直接从官网下载即可
　 　llvm源码：[http://llvm.org/releases/3.8.0/llvm-3.8.0.src.tar.xz](http://llvm.org/releases/3.8.0/llvm-3.8.0.src.tar.xz)
　 　CLang源码：[http://llvm.org/releases/3.8.0/cfe-3.8.0.src.tar.xz](http://llvm.org/releases/3.8.0/cfe-3.8.0.src.tar.xz)

## 3. 编译过程

1. 打开cmd，cd到解压后的目录。运行 mkdir build，新建一个build文件夹，这样是为了防止编译内容和源码混乱，以后方便维护。

    ``` shell
    E:\SourceCode\llvm\llvm-3.8.0.src>mkdir build
    E:\SourceCode\llvm\llvm-3.8.0.src>cd build
    E:\SourceCode\llvm\llvm-3.8.0.src\build>
    ```

2. 用CMake处理

    ``` shell
    E:\SourceCode\llvm\llvm-3.8.0.src\build>cmake 
                    -G "MinGW Makefiles"
                -DCMAKE_BUILD_TYPE=Release 
                -DCMAKE_INSTALL_PREFIX=D:\LLVM  
                -DCMAKE_MAKE_PROGRAM=mingw32-make.exe
                ..
    ```

    为了方便说明，我把参数拆开，实际用的时候写一行即可。
    G为要生成的格式，用MinGW，这里大小写必须写对，写错的话会有提示。
    CMAKE_BUILD_TYPE为构建类型，可以写Debug，Release，MinSizeRel。
    CMAKE_INSTALL_PREFIX为install路径，一般这里路径最好保守一些，尽量不要用中文或空格。
    CMAKE_MAKE_PROGRAM这里写不写都可以。
    最后的两点指的是上层路径，按CMake文档，应该写在前面，实际写到后面好像也没什么。

3. 编译LLVM。前面的几步运行的还算快，make就是挑战耐性的时刻了，MinGW用的是mingw32-make，这里很简单，但是是最耗时的，休息一下。这里完成以后，install一下，就能放到CMake设置里的位置了

    ``` shell
    E:\SourceCode\llvm\llvm-3.8.0.src\build>mingw32-make
    …………
    E:\SourceCode\llvm\llvm-3.8.0.src\build>mingw32-make install
    ```

4. 配置编译Clang，步骤和编译LLVM差不多，不细说了，直接上参数

    ``` shell
    E:\SourceCode\llvm\cfe-3.8.0.src>cmake 
                -G "MinGW Makefiles" 
                -DCMAKE_BUILD_TYPE=Release 
                -DCMAKE_INSTALL_PREFIX=D:\LLVM
                ..
    E:\SourceCode\llvm\cfe-3.8.0.src\build>mingw32-make
    E:\SourceCode\llvm\cfe-3.8.0.src\build>mingw32-make install
    ```

5. 最后不要忘了把编译输出的路径加到Path里面

## 4. 测试

打开cmd，输入 clang -v 输出版本

``` shell
C:\Users\wjh>clang -v
clang version 3.8.0 (tags/RELEASE_380/final)
Target: x86_64-w64-windows-gnu
Thread model: posix
InstalledDir: D:\LLVM\bin
```

下面用程序测试 test.cpp

``` cpp
#include <iostream>
using namespace std;
int main()
{
    cout << "Hello World!" << endl;
    return 0;
}
```

编译

```shell
E:\test>clang++ -o test.exe test.cpp
```

运行

```shell
E:\test>test.exe
Hello World!
```

## 5. 编译问题或其他

* libcxx和libcxxabi在Windows上编译总有问题，没成功，以后再试，如果有编译成功的望指教。
* 太穷，买不起Mac，有时间还是在Linux上试试，移植的东西肯定没原装的用着顺。
