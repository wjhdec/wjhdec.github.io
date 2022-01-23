# java8升级到java11的模块系统


:sectnums:
:icons: font

== 升级背景

模块化: 指的是从 java9 开始的 https://www.runoob.com/java/java9-module-system.html[java-module]  https://openjdk.java.net/projects/jigsaw/[Jigsaw]

=== 升级相关情况及目的

1. 新增功能更方便的模块化
2. 模块系统中可以使用 jlink 压缩java环境，减小最终体积

TIP: Java11 为长期支持版本，理论会比其他版本稳定。

=== 升级前环境

1. java8环境
2. gradle管理依赖
3. Springboot环境
4. 工程分布在多个jar包中
5. 有混淆逻辑

== 注意事项

=== 模块化

过程很简单，也很复杂，在java根目录加上 `module-info.java`，再在里面加上依赖及发布内容。
需要注意，网上一些（一大部分）添加 module-path 的方式是遍历task，只要有编译的就加上 module-path。这种方式会让单元测试报找不到 module-path 的错误，解决方式是只在 compileJava 中添加 module-path。

=== 模块化中的依赖

其实这也是升级前的最初想法，以前依赖过重，不同版本重复jar包过多，甚至还有各种魔改jar包，有时会出现奇怪的玄学问题。

模块系统对依赖特别严格，升级前必须把工程的依赖关系最简化，这关系到原有程序能不能升级为模块系统。

由于升级前的工程需要验证kerberos，再加上hive等的客户端验证，需要依赖hadoop。而hadoop的依赖只有你想不到的……，查看依赖，只算各式各样的jackson就不下五六种。
因此解决依赖分为了几步：

1. 重新设计kerberos的客户端，去掉对hdfs包里面的用户验证的依赖
2. 使用webhdfs代替hdfs，直接使用rest服务访问而不是使用jar包，去掉了对 hdfs 的依赖
3. 连接hive的driver改为反射读取，去掉直接的工程内部依赖
4. 由于以前多个子工程使用了很多相同的包路径，这在非模块系统中很方便，但是模块系统中反而是个麻烦

[NOTE]
====
模块系统中，如果多个依赖 module（jar包）中有相同的 package，且相同package下都有需要对外发布的class，会报冲突。如果想解决，只能自己再封装jar包，比较麻烦。

eg. +
    a.jar 中包含 package `c.d.e` ，且此包中有 `f.class` +
    b.jar 中也包含 package `c.d.e` ，且此包中有 `g.class` +
如果一个工程同时依赖 a.jar 和 b.jar，则会报冲突。比较常见的解决方法就是把 a.jar 和 b.jar 合并为一个jar包。
====

经过一系列修改后完成了依赖的最简化，实现了引用的不冲突。

=== 混淆

这次的升级顺便修改了混淆的思路，以前是各个子工程不混淆，最后一个门户工程再统一打包混淆；本次升级直接混淆各个子工程，门户工程反而因为没啥具体内容，可以不再混淆。

这样的改变其实和模块化有一定关联，模块化中需要在 `module-info.java` 中指明外部能够访问的内容，且各个模块不能再相同的package下写可执行的内容。如果放到最后再混淆会有以下几个问题：

1. 每次新添加模块都要考虑修改混淆脚本（路径不同）
2. 不能动态的添加模块（每次新加模块都要重新混淆工程）

TIP: 每个模块里面对外发布的内容可以都写成接口，混淆时规定接口不混淆。不准备对外的类可以实现接口，然后添加该类的 Builder 类，暴露出不混淆的接口和 Builder 即可。

==== gradle 发布到 nexus 技巧（maven-publish 插件）

1. 混淆前的jar包打包及发布逻辑不动
2. 混淆后的内容生成jar包时在名称中添加自定义的 classifier，用 artifact 方式和原始jar包一起发布到nexus服务
3. 使用的工程用 `compile 'group:name:version:classifier'`，引入依赖，就是在常用的依赖后面加上冒号+自定义的classifier名称

=== jlink

这里是比较难的一块，大概用了两天时间才完成，使用的是 `org.beryx.jlink` 插件，它能够把工程中非module的包打包成一个（自动执行上面提到的合并jar包过程），然后根据依赖生成精简后的jdk。这里难点有几个：

1. 像是springboot这种结构比较分散的不好确定依赖，我的流程基本是 `测试 -> 看错误信息 -> jlink中添加依赖`，这样的流程重复了几十遍。
2. 打包速度慢，一次大概得几分钟

=== docker打包

这里原以为很简单，实际又跳坑里了。由于jlink打包了jdk，最开始使用的是 `alpine` 作为基础，但是一直报找不到 java 的错误，后来查资料才知道是alpine中缺少环境。改用 `alkoclick/alpine-jdk11-nojdk:latest`，成功完成打包。

== 成果

经过一系列的调整，最直接可见的成果是最后打包docker的体积减小了约60%

