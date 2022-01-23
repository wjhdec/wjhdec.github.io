# Scrapy 爬虫框架初试

　 　以前用python写过一些简单的爬虫，没用过框架。一个朋友要爬取api数据，数据量不算太大，不过得用一些递归爬取，这次趁这个机会尝试一下用框架爬取数据。

<!-- more -->

## 1. Scrapy部署

### 1.1. 环境信息

　 　用 windows 系统搭建scrapy环境跟麻烦，由于scrapy需要依赖twisted，而twisted只有源码编译，如果系统安装时没有合适的编译环境，则不能正常安装。
　 　于是选择linux作为系统环境，比较方便搭建。环境详情如下：

| 内容   | 版本                          |
|:-------|:------------------------------|
| 系统   | CentOS Linux release 7.1.1503 |
| python | Python 2.7.5                  |

### 1.2. 安装配置

#### 1.2.1. 安装pip

　 　默认centos中的python是不安装pip的，为了方便，先安装pip

``` bash
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
```

　 　安装完成后用 -V 确认

``` bash
pip -V
```

#### 1.2.2. 安装其他环境

　 　由于需要编译及其他需要，还要安装其他内容

``` bash
yum install openssl-libs openssl-devel openssl gcc
```

#### 1.2.3. 安装scrapy

``` bash
pip install scrapy
```

　 　这里步骤很简单，但是很容易出错

#### 1.2.4. 安装时易出的问题

　 　如果twisted未正常安装，需要手动安装时，要注意不要安装最新版本，否则不能正常运行。我在这里用的是16.4.1版本

``` bash
pip install twisted==16.4.1
```

## 2. 编写爬取数据逻辑

### 2.1. 建立工程

``` bash
scrapy startproject spider
```

　 　这时，创建了一个名为spider的工程，里面已经有了结构

![目录结构](./2017_03_10_scrapyTest/project_files.png)

### 2.2. 查看网页结构

　 　由于这次是爬取 ArcGIS的api，形势比较规整，提取较为容易。

![布局](./2017_03_10_scrapyTest/html_layout.png)

![爬取布局](./2017_03_10_scrapyTest/html_layout2.png)

### 2.3. 创建爬取逻辑

#### 2.3.1. 添加 item

　 　在./spider/spider/items.py 中创建item，这里主要目的是记录结构

![items](./2017_03_10_scrapyTest/code_item.png)

#### 2.3.2. 创建主文件

　 　在./spider/spiders中创建文件，并编写逻辑

![代码结构](./2017_03_10_scrapyTest/code_demo.png)

源码详见 [github](https://github.com/wjhdec/spiders/blob/master/spider/spider/spiders/wpf.py)

#### 2.3.3. 配置输出

　 　这里需要设置 ./spider/spider/pipelines.py

``` python
class ArcgiswpfPipeline(object):
    def open_spider(self, spider):
        self.file = open('items.line-json', 'wb')
    def process_item(self, item, spider):
        line = json.dumps(dict(item)) + "\n"
        self.file.write(bytes(line, encoding = "utf8"))
        return item
    def spider_closed(self, spider):
        self.file.close()
```

　 　主要是指定输出格式，这里输出成了json，按行输出。

#### 2.3.4. 其他注意点

　 　这里有几个注意点，方法中 def parse(self, response): 为入口方法，这里主要用了xpath和regex的方式去获取数据。另外，还要遍历左侧的树形列表，这里建立一个set记录已经处理的页，处理过的则不处理。
　 　这里也有xpath的一些技巧，比如需要获取一个节点的全部文字内容，则可以用 m.xpath(“string(.)”).extract()[0]获取。

## 3. 其他优化

　 　由于ArcGIS在国内服务器访问比较慢，这里需要应用代理。好在scrapy直接能设置代理。
　 　配置一下 ./spider/spider/middlewares.py

　 　在已有的类里面添加方法

``` python
def process_request(self, request, spider):
    # Set the location of the proxy
    request.meta['proxy'] = "http://192.168.1.6:8087"
```

　 　在 ./spider/spider/setting.py 中把 DOWNLOADER_MIDDLEWARES 解除注释，并改成刚改过的类。

## 4. 爬取数据

``` bash
scrapy crawl arcgis_wpf
```

![处理数据](./2017_03_10_scrapyTest/demo.png)

　 　结果数据(1条)

```json
{
    "class_name": "SuggestParameters",
    "class_props": [
        {
            "type": "Public method",
            "name": "SuggestParameters",
            "desc": "Creates a new SuggestParameters instance."
        },
        {
            "type": "Public property",
            "name": "Categories",
            "desc": "Gets a mutable list of categories from which addresses should be returned."
        },
        {
            "type": "Public property",
            "name": "CountryCode",
            "desc": "Gets or sets a value representing the country. Providing this value increases geocoding speed. Acceptable values include the full country name, the ISO 3166-1 2-digit country code, or the ISO 3166-1 3-digit country code. A list of supported countries and codes is available here."
        },
        {
            "type": "Public property",
            "name": "MaxResults",
            "desc": "Gets or sets the maximum number of returned suggestions."
        },
        {
            "type": "Public property",
            "name": "PreferredSearchLocation",
            "desc": "Gets or sets preferred search location."
        },
        {
            "type": "Public property",
            "name": "SearchArea",
            "desc": "Gets or sets the geographic area for which searching for locations will be limited to."
        }
    ],
    "class_desc": "A class representing the parameters to a geocoding suggestion operation.",
    "namespace": "Esri.ArcGISRuntime.Tasks.Geocoding",
    "class_type": "class"
}
```

　 　好了，到这里就结束了。得到这些数据后可以再解析，得到想要的结构。
