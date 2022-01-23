# 在 golang 中使用 GDAL（Windows环境）


今天下午才说windows下的go语言不能调用 GDAL，晚上回来看一下源码，发现可以在Windows下使用。写下来记录一下。

<!-- more -->

## 1. 背景说明

以前都是使用的C++作为开发语言，GDAL开发没问题；但是在Windows环境中验证Kerberos时遇到了问题，暂时没有解决方案。以前用 golang 连接Kerberos验证成功过，于是准备看一下是否Golang能直接使用GDAL。

使用的是 [lukeroth/gdal](https://github.com/lukeroth/gdal) 库

在今天下午看这个工程时，先查看的 issue，搜索Windows，只有几篇提到了，也没明确说可以用。主工程的说明文档中之提到了 Ubuntu 中可以使用。

晚上再看源码时发现了这个工程有 [c_windows_amd64.go](https://github.com/lukeroth/gdal/blob/master/c_windows_amd64.go) 这个文件，说明是可以在Windows下使用的。准备编译测试一下。

## 2. 编译

### 2.1 mingw64 环境

根据上面那个文档，可以看出这个工程使用了 cgo，依赖于gcc，于是使用了以前搭建的 [MSYS2](https://www.msys2.org/) 环境

msys2 安装方式略

直接安装 GDAL：

`pacman -S mingw-w64-x86_64-gdal `

### 2.2 创建测试工程

创建工程略，测试代码如下

```go
package main

import (
	"github.com/lukeroth/gdal"
	"log"
)

func main() {
	ds := gdal.OpenDataSource("E:\\Data\\shjq\\sh_wgs84.shp", 0)
	defer ds.Destroy()
	layer := ds.LayerByIndex(0)
	for{
		if feature := layer.NextFeature(); feature != nil{
			defer feature.Destroy()
			geom := feature.Geometry()
			wkt, err := geom.ToWKT()
			if err != nil {
				log.Fatalf("to wkt error: %s", err)
			}
			log.Println(wkt)
		}else {
			break
		}
	}
}

```



### 2.3 编译

直接在命令行运行 `go build` 会出现找不到 `gdal_i` 的问题，这时因为在 c_windows_amd64.go 文件中写死了找包的位置，需要修改源码。

这里我直接改的  `C:\Users\{USER_NAME}\go\pkg\mod\github.com\lukeroth\gdal@v0.0.0-xxxx` 里面的 c_windows_amd64.go



第6，7行改成mingw64为基础的路径（这里文件默认是只读，需要在属性里面把只读去掉）。注意把 `-lgdal_i` 中的 `_i` 去掉。

```go
/*
#cgo windows LDFLAGS: -LD:/msys64/mingw64/lib -lgdal
#cgo windows CFLAGS: -ID:/msys64/mingw64/include
*/
```

此时再执行 go build 即可通过



## 3. 存在问题

使用过程中发现很多问题：

1. 说明中写的是支持 GDAL2，其实使用 GDAL 3 也能正常编译（不知道有没有坑）；
2. 这个工程虽然近期有更新，但是更多的仍然是基于 GDAL 2 的接口，GDAL3中 Dataset 的接口及其不完整，仍然需要使用 DataSource 接口的相关方法；
3. 用msys2中的GDAL依赖过多，需要的 dll 太多太乱，有时间直接用源码最简编译试试；
4. 建议直接改源码的查找路径，我试过把lib文件和include文件夹放到源码中写死的目录下，但是编译失败；
5. 示例太少，需要看源码中资源的释放情况，有的源码中有资源释放，有的需要手动执行释放。



这个工程能不能满足需求还需进一步测试

