# Windows系统上spark环境搭建


　 　今天测试一下在window上搭建spark环境，此环境只适用于代码测试，实际运行要在集群环境下。

<!-- more -->

## 基本环境

　 　我在自己的个人电脑上搭建，环境为 win10 家庭版，内存12G

### 注意事项

1. 本环境搭建目的是测试代码，不做生产环境。
2. 计算机环境内存要够用，不然测试都带不起来。
3. 本环境没有hdfs，只读取本地文件

## 环境搭建

### 下载winutils

　 　spark需要依赖hadoop的部分内容，github上有winutils文件下载
　 　[winutils下载](https://github.com/steveloughran/winutils)
　 　这个文件不大，大概4.7M，可以都下载下来。

### 配置winutils

　 　把压缩文件中的 hadoop-2.7.1 解压到某个位置(我用的是这个版本，也可以用其他的，这个和等会儿要下载的spark预编译文件有关系)。
　 　添加环境变量，在windows里添加 HADOOP_HOME 的环境变量，指向刚才复制的文件路径

![hadoop环境](./2017-08-13-sparkOnWindows/1_hadoop_env.png)

### 下载spark

![spark下载url](./2017-08-13-sparkOnWindows/1_download_spark.png)

　 　解压文件到某个地址

### spark测试

　 　打开cmd，cd到spark的bin目录，执行spark-shell。出现spark版本信息，说明配置基本成功。

![运行信息](./2017-08-13-sparkOnWindows/2_spark-version.png)

## 程序测试

### 编写Scala程序

　 　此程序段用scala编写，仅供测试；步骤是读取英文小说( 传说中的《时间简史》)，通过空格和标点符号分词，然后进行词频处理，词频排序最后进行输出。

``` scala

package indi.wjh.spark.testenv

import org.apache.spark.SparkConf
import org.apache.spark.sql.{SaveMode, SparkSession}

object SparkMain {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder().config(new SparkConf()).getOrCreate()
    import spark.implicits._
    val wcount = spark.read.text("D:\\txt\\A Brief History of Time.txt")
      .as[String].flatMap(_.split("[\\s+|\\p{P}]")).filter(_.trim != "")
      .groupByKey(_.toLowerCase).count()
    val wordOrder = wcount.orderBy(wcount(wcount.schema.fields(1).name).desc)
    //输入成1个文件
    wordOrder.repartition(1).write.mode(SaveMode.Overwrite)
      .csv("D:\\txt\\count.csv")
    wordOrder.show()
    spark.stop()
  }
}
```

　 　执行命令 run.bat

``` bash
D:\services\spark-2.0.2-bin-hadoop2.7\bin\spark-submit2 ^
    --master local[*] ^
    --name wcount ^
    --class indi.wjh.spark.testenv.SparkMain ^
    testenv-1.0-SNAPSHOT.jar
```

![运行中](./2017-08-13-sparkOnWindows/words_value.png)

![完成](./2017-08-13-sparkOnWindows/words_result.png)

完成输出，共 8119行

**注意：**spark在windows上有bug，不能删除临时文件，这个问题暂时没解决，需手动删除。
