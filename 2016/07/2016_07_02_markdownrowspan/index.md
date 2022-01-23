# Markdown导出html时rowspan与colspan的处理



　　MarkDwon原生不支持table，用扩展的table很简单方便，但是这个table不支持单元格合并。用html可以写出需要的表格，但是写的内容较多且不易维护。

<!-- more -->

　　下面用sublime的OmniMarkupPreviewer插件实现rowspan及colspan。

## 1. 实现思路

　　这里利用的是sublime的OmniMarkupPreviewer插件，利用它导出html。OmniMarkupPreviewer有一个特点是能够运行写在md文件里的javascript，可以利用js对html进行dom操作。

　　试过很多markdown工具进行html生成，都不能很好的解析js。所以下面的这个方法并不是通用方法。只适用于利用md文件生成html。

## 2. 实现过程

　　sublime和OmniMarkupPreviewer安装使用就不说了，网上教程很多。下面内容只是实现。

### 2.1. 添加js脚本

　　原理是在html的td中搜索指定标示，然后进行rowspan或colspan的属性填充，最后删除标示。
　　标示有：

* rowspan(n) 纵向合并
* colspan(n) 横向合并
* deltd 删除td，rowspan或colspan覆盖的地方需要用，不然会挤出

　　添加代码如下：

``` javascript
document.onreadystatechange = function () {   
     if(document.readyState=="complete") {
        var rowspanpat = /##rowspan\(\d*\)##/g;
        var deltdpat = /##deltd##/g;
        var colspanpat = /##colspan\(\d*\)##/g;
        var tds = document.getElementsByTagName("td");
        for(var i= 0 ;i<tds.length;i++){
            var deltdmatch = tds[i].innerHTML.match(deltdpat);
            if(deltdmatch && deltdmatch[0]){
                tds[i].parentNode.removeChild(tds[i]); 
                i--;
            }else{
                span(tds[i], "rowspan");
                span(tds[i], "colspan");
            }
        }
      }
}
function span(td, spantype){
    var spanpat = null;
    if(spantype == "rowspan"){
        spanpat = /##rowspan\(\d*\)##/g;
    }else{
        spanpat = /##colspan\(\d*\)##/g;
    }
    var match = td.innerHTML.match(spanpat);
    if(match && match[0]){
        td.innerHTML = td.innerHTML.replace(match[0], '')
        var spanval = match[0].match(/\d+/g);
        td.setAttribute(spantype, spanval);
    }
}
```

### 2.2. 实例效果

js脚本加入到md文件最下面

``` html
# 测试
| 测试            | 测试1     | 测试2           | 测试3 |
|:----------------|:----------|:----------------|:------|
| a               | b         | c##rowspan(2)## | d     |
| e##colspan(2)## | ##deltd## | ##deltd##       | f     |

<script type="text/javascript">
    document.onreadystatechange = function () {   
         if(document.readyState=="complete") {
            var rowspanpat = /##rowspan\(\d*\)##/g;
            var deltdpat = /##deltd##/g;
            var colspanpat = /##colspan\(\d*\)##/g;
            var tds = document.getElementsByTagName("td");
            for(var i=0;i<tds.length;i++){
                var deltdmatch = tds[i].innerHTML.match(deltdpat);
                if(deltdmatch && deltdmatch[0]){
                    tds[i].parentNode.removeChild(tds[i]); 
                    i--;
                }else{
                    span(tds[i], "rowspan");
                    span(tds[i], "colspan");
                }
            }
          }
    }
    function span(td, spantype){
        var spanpat = null;
        if(spantype == "rowspan"){
            spanpat = /##rowspan\(\d*\)##/g;
        }else{
            spanpat = /##colspan\(\d*\)##/g;
        }
        var match = td.innerHTML.match(spanpat);
        if(match && match[0]){
            td.innerHTML = td.innerHTML.replace(match[0], '')
            var spanval = match[0].match(/\d+/g);
            td.setAttribute(spantype, spanval);
        }
    }
</script>
```
