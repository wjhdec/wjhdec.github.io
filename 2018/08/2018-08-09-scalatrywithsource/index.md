# scala try with resource 方法


　　scala中没有原生的类似 java 7+ 中的 try(){}catch{} 自动关闭资源的方法。想用的话只能自己写了。
　　PS: 以前没注意过，shapeless功能真很强大。

<!-- more -->

　　资料不好找啊，网上大部分能搜索到的方法是 参考1 中的方法，比较直接，但是有一个问题，不能实现同时对多个资源close。后来努力寻找，翻到了 参考2 ，不过也存在一些问题，例如在资源关闭时出错的话会直接忽略。
　　于是整合了一下，有了以下的逻辑：

　　**注意:** 单个和多个必须分开处理

```scala
package per.wjh.utils

import shapeless._
import ops.hlist._
import ops.nat._

import scala.util.control.NonFatal
import scala.util.{Failure, Try}

object TryWith {

  def apply[C <: AutoCloseable, R](resource: => C)(f: C => R): Try[R] =
    Try(resource).flatMap(resourceInstance => {
      try {
        val returnValue = f(resourceInstance)
        Try(resourceInstance.close()).map(_ => returnValue)
      } catch {
        case NonFatal(exceptionInFunction) =>
          try {
            resourceInstance.close()
            Failure(exceptionInFunction)
          } catch {
            case NonFatal(exceptionInClose) =>
              exceptionInFunction.addSuppressed(exceptionInClose)
              Failure(exceptionInFunction)
          }
      }
    })

  def apply[C, Len <: Nat, L <: HList, R](resources: => C)(f: C => R)(
    implicit gen: Generic.Aux[C, L],
    con: ToList[L, AutoCloseable],
    length: Length.Aux[L, Len],
    gt: GT[Len, nat._1]
  ): Try[R] = {
    Try(resources).flatMap(resourceInstance =>
      try {
        val returnValue = f(resources)
        Try(gen.to(resourceInstance).toList.reverse.foreach(_.close()))
          .map(_ => returnValue)
      } catch {
        case NonFatal(exceptionInFunction) =>
          try {
            gen.to(resourceInstance).toList.reverse.foreach(_.close())
            Failure(exceptionInFunction)
          } catch {
            case NonFatal(exceptionInClose) =>
              exceptionInFunction.addSuppressed(exceptionInClose)
              Failure(exceptionInFunction)
          }
      }
    )
  }
}
```

　　下面简单测试一下：

``` scala
package per.wjh.utils.test

import java.io.Closeable

import org.scalatest.FunSuite
import per.wjh.utils.TryWith

import scala.util.{Failure, Success}

class TryTest extends FunSuite {

  case class Closeable1() extends Closeable {
    def info(): Unit = {
      println("print 1")
    }

    override def close(): Unit = {
      println("close 1 ")
    }
  }

  case class Closeable2() extends Closeable {
    def info(): Unit = {
      throw new Exception("error in 2")
      println("print 2")
    }

    override def close(): Unit = {
      println("close 2 ")
    }
  }

  test("测试Try") {
    TryWith(Closeable1(), Closeable2()) { case (a, b) =>
      a.info()
      b.info()
      "success"
    } match {
      case Success(value) => println(value)
      case Failure(e) => println(e.getMessage)
    }
    println("#######################")
  }

  test("测试Try2") {
    TryWith(Closeable2()) { a =>
      a.info()
      "success"
    } match {
      case Success(value) => println(value)
      case Failure(e) => println(e.getMessage)
    }
    println("#######################")
  }
}

```

　　测试结果：

```log
print 1
close 2 
close 1 
error in 2
#######################
close 2 
error in 2
#######################

```

　　Closeable 继承自 AutoCloseable，好玩儿吧？这里用 AutoCloseable适用性更广。
还有，参考2中的**关闭顺序**有问题，我在上面的新代码中添加了倒序处理。

参考：

1. <https://gist.github.com/dmyersturnbull/37f85b7703ea07f5cf88d614a32c98b8>
2. <https://dzone.com/articles/simple-try-with-resources-construct-in-scala>
