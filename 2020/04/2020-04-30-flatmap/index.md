# flatMap 在 Scala(Spark)/Java8 中的使用




flatMap 是个很神奇的东西，一般是学习 scala 的第一个坎儿，其实很简单，下面分别从 Scala，Spark（Dataset），Java8（lambda）说明flatMap的使用

<!-- more -->

## 1. 在 Scala 中的使用

flatMap 在scala的参数需要的返回是 GenTraversableOnce，下面是实现：

```scala
def flatMap[B, That](f: A => GenTraversableOnce[B])(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    def builder = bf(repr) // extracted to keep method size under 35 bytes, so that it can be JIT-inlined
    val b = builder
    for (x <- this) b ++= f(x).seq
    b.result
}
```



GenTraversableOnce（遍历器？不知道怎么翻译）包含最基础的遍历方法，Iterable，Seq，List等都是继承自这里，下面是比较有意思的一点：

### 1.1. 过滤 None 值

Scala里的Optional，很重要的一句，把Option隐式转换为 Iterable 

```scala
object Option {
    // ......
  /** An implicit conversion that converts an option to an iterable value
   */
  implicit def option2Iterable[A](xo: Option[A]): Iterable[A] = xo.toList
    // ......
}
```



再看 toList 的实现：



```scala
/** Returns a singleton list containing the $option's value
   * if it is nonempty, or the empty list if the $option is empty.
   */
def toList: List[A] =
  if (isEmpty) List() else new ::(this.get, Nil)
```



Optional 可直接转换为 Iterable，如果为空，返回空数组，如果不是则返回单个值，很神奇的操作……

于是有了下面的操作

```scala
class FlatMapTest extends AnyFlatSpec with Matchers {
  it should "过滤空值" in {
    val result = Seq(1, 2, 3, 4, 5, 6, 7, 8).flatMap { v =>
      if(v % 2 == 0){
        Some(v)
      }else {
        None
      }
    }
    result should be(Seq(2,4,6,8))
  }
}
```



### 1.2. 正常 flatMap 操作



很正常的操作，把内部数组向上提

示例：

```scala
class FlatMapTest extends AnyFlatSpec with Matchers {
  it should "真.flatMap" in {
    val result = Seq("a.b","c.d").flatMap(_.split("\\."))
    result should be(Seq("a", "b", "c", "d"))
  }
}
```



## 2. 在spark（Dataset）中使用

基于 Optional 的神奇特性，spark 中 Dataset 的 flatmap也有类似功能

下面是spark的 flatMap 需要的返回值是 TraversableOnce 这是 GenTraversableOnce 的子类，使用上面的经验完全没问题。

```scala
/**
 * :: Experimental ::
 * (Scala-specific)
 * Returns a new Dataset by first applying a function to all elements of this Dataset,
 * and then flattening the results.
 *
 * @group typedrel
 * @since 1.6.0
 */
@Experimental
@InterfaceStability.Evolving
def flatMap[U : Encoder](func: T => TraversableOnce[U]): Dataset[U] =
  mapPartitions(_.flatMap(func))
```



直接上示例：

```scala
import org.apache.spark.sql.SparkSession
import org.junit.runner.RunWith
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatestplus.junit.JUnitRunner

case class Demo(name: String, inUse: Boolean)

@RunWith(classOf[JUnitRunner])
class FlatMapTest extends AnyFlatSpec with Matchers {
  private val spark = SparkSession.builder().master("local").getOrCreate()
  import spark.implicits._
  private val ds = spark.createDataset(Seq(
    Demo("a.b", inUse = true),
    Demo("c.d", inUse = false)))

  it should "过滤用" in {
    val result = ds.flatMap{d =>
      if(d.inUse){
        Some(d)
      }else{
        None
      }
    }
    result.collect().head should be (Demo("a.b", inUse = true))
  }

  it should "真.flatMap" in {
    ds.flatMap( d => d.name.split("\\.")).collect() should be(Seq("a", "b", "c", "d"))
  }

}

```



## 3. 在 Java8 中的使用

其实在 Java8 中没有这么花式的玩法，Option有单独的 flatMap （scala的Option其实也有flatMap，不细说了）

### 3.1. steam 中的 flatMap

方法：

```java
package java.util.stream;

<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
```

返回值仍然是流，比较简单

示例：

```java
@Test
public void testStream(){
    List<List<String>> demo = Arrays.asList(
        Arrays.asList("a","b"), 
        Arrays.asList("c","d"));
    List<String> result = demo.stream()
        .flatMap(Collection::stream)
        .collect(Collectors.toList());
    result.forEach(System.out::print);
}
```



### 3.2. Optional 中的 flatMap

方法：

```java
package java.util;
public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Objects.requireNonNull(mapper.apply(value));
    }
}
```

这个更简单，基本没啥说的

示例：

```java
@Test
public void testOptional(){
    Optional<String> demo1 = Optional.of("demo");
    Optional<String> result1 = demo1.flatMap(d -> Optional.of(d + " is not null"));
    Assert.assertEquals(result1.orElse(""), "demo is not null");

    Optional<String> demo2 = Optional.empty();
    Optional<String> result2 = demo2.flatMap(d -> Optional.of(d + " is not null"));
    Assert.assertEquals(result2.orElse(""), "");
}
```

好处是可以嵌套使用

## 4. 结语

就到这里吧，最近状态不太好，正在纠结些问题，反正水帖子，能看就行
