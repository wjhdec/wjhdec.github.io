# Spark中多个Dataset进行union操作的方法


　　开年第一篇，写个简单点的。spark中多个Dataset进行union的方式，这种情况并不是太多见，不过也不算少，例如合并多个按时间保存的数据。

<!-- more -->

## 1. 不是环境的环境

| 项目          | 内容             |
|:--------------|:-----------------|
| 系统          | win10家庭版      |
| spark源码版本 | v2.0.2           |
| IDE           | IDEA社区版       |
| 配置          | 太低，不好意思说 |
| 其他          | 还没想好         |

　　能测试就行，凑合用

## 2. 搭建测试环境

　　用gradle托管，最基础的包即可

``` groovy
group 'pers.wjh.spark.units'
version '1.0-SNAPSHOT'

apply plugin: 'scala'

sourceCompatibility = 1.8

repositories {
    maven {url "http://maven.aliyun.com/nexus/content/groups/public"}
}

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
    compile "org.apache.spark:spark-core_2.11:2.0.2"
    compile "org.apache.spark:spark-sql_2.11:2.0.2"
}
```

## 3. 多个Dataset的union方式

### 3.1. 基础union

　　适合于已知要union的Dataset的个数，简单直接。

``` scala
val dsAll = ds1.union(ds2).union(ds3)
```

### 3.2. 递归union

　　利用Dataset接口的union方法，进行递归操作

``` scala
/**
  * 递归union执行
  * @param dsArray 要union的列表
  * @param index 位置
  * @return
  */
private def recurUnionGroup(dsArray: Seq[Dataset[Row]], index:Int):Dataset[Row]=
  if (index < dsArray.length-1) dsArray(index).union(recurUnionGroup(dsArray,index+1)) else dsArray.last
```

　　这里用递归的方式实现上种方法中的连续union，不过这种方法已经可以实现动态的union，虽然方法比较直接。

### 3.3. 调用Union直接union

　　重头戏来了，其实spark里面已经有了多个Dataset进行union的方法，只不过藏的稍微深了点，先上代码实现：

``` scala
/**
  * 调用Union方法
  * @param dsArray
  * @return
  */
def unionGroup(dsArray: Seq[Dataset[Row]]):Dataset[Row] =
  new Dataset(spark,Union(dsArray.map(_.queryExecution.logical)),RowEncoder(dsArray.head.schema))
```

　　我们可以先看Dataset接口实现的union源码,位置在：
　　*spark-2.0.2\sql\core\src\main\scala\org\apache\spark\sql\Dataset.scala*
　　第 1459 行

``` scala
def union(other: Dataset[T]): Dataset[T] = withSetOperator {
  // This breaks caching, but it's usually ok because it addresses a very specific use case:
  // using union to union many files or partitions.
  CombineUnions(Union(logicalPlan, other.logicalPlan))
}
```

　　这里调用了Union方法，其实Union还有一个实现,可以执行多Dataset的union

``` scala
/** Factory for constructing new `Union` nodes. */
object Union {
  def apply(left: LogicalPlan, right: LogicalPlan): Union = {
    Union (left :: right :: Nil)
  }
}

case class Union(children: Seq[LogicalPlan]) extends LogicalPlan {
  override def maxRows: Option[Long] = {
    if (children.exists(_.maxRows.isEmpty)) {
      None
    } else {
      Some(children.flatMap(_.maxRows).sum)
    }
  }
  // 这里省略一些......
}
```

　　不过也可以看到，这里的参数是LogicalPlan，而不是Dataset，由于只是Dataset的union，我们之间找到dataset初始化时的LogicalPlan即可，LogicalPlan在Dataset里是私有字段，只好去找初始化时的queryExecution了。
　　Union完成后得到了新的LogicalPlan，再用这个LogicalPlan执行生成新的Dataset，即为需要的union完成后的Dataset。

## 4. 测试

　　用单元测试进行简单的显示：

``` scala
@Test
  def testRecurUnion():Unit={
    println("递归union结果：")
    val dsList = Seq(ds1,ds2,ds3)
    val multiUnion = MultiUnion(spark)
    multiUnion.recurUnionGroup(dsList).show()
  }
  @Test
  def testBase():Unit={
    println("依次union结果：")
    val dsAll = ds1.union(ds2).union(ds3)
    dsAll.show()
  }

  @Test
  def testUnion():Unit={
    println("调用union类结果：")
    val dsList = Seq(ds1,ds2,ds3)
    val multiUnion = MultiUnion(spark)
    multiUnion.unionGroup(dsList).show()
  }
```

　　输出结果：

``` bash
spark方法调用union结果：
+---+-----+
| id|value|
+---+-----+
|  1|    a|
|  2|    b|
|  3|    c|
|  4|    d|
|  5|    e|
|  6|    f|
+---+-----+

依次union结果：
+---+-----+
| id|value|
+---+-----+
|  1|    a|
|  2|    b|
|  3|    c|
|  4|    d|
|  5|    e|
|  6|    f|
+---+-----+

递归union结果：
+---+-----+
| id|value|
+---+-----+
|  1|    a|
|  2|    b|
|  3|    c|
|  4|    d|
|  5|    e|
|  6|    f|
+---+-----+
```

## 5. 其他说明

　　由于spark都是lazy执行，这几种方法在执行效率上差异并不明显。

　　源码详见 ：[Union类](https://github.com/wjhdec/sparkutils/blob/master/core/src/main/scala/pers/wjh/bigata/utils/core/union/MultiUnion.scala) 及 [测试](https://github.com/wjhdec/sparkutils/blob/master/core/src/test/scala/pers/wjh/bigata/utils/core/test/UnionTest.scala)

