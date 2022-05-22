# Restful 风格接口设计总结


## 1. 背景

最近在维护一个非 Restful 接口风格的项目。由于习惯了设计 Restful 风格的接口，每当在这个项目中添加接口时就特别纠结，总有种无从下手的感觉。或许我有强迫症？？ &#128579; 趁这个感觉很别扭的机会总结一下设计思路

为了方便描述，先假设一个场景，现在有班级和学生两个表，要对这两个表设计接口



班级表：

| id   | name |
| ---- | ---- |
| 1    | 一班 |
| 2    | 二班 |



学生表：

| id   | class_id | name | age  |
| ---- | -------- | ---- | ---- |
| 1    | 1        | 甲   | 10   |
| 2    | 1        | 乙   | 11   |
| 3    | 2        | 丙   | 10   |
| 4    | 2        | 丁   | 11   |



## 2. Restful 风格接口核心设计思想

### 2.1. 一切以 <u>**资源**</u> 为核心

Restful 风格的设计中，全部的操作都是围绕  `资源`  进行的，这是最中心的思想。
例如上面的例子中，资源有2个，班级和学生。那么接下来的设计就需要从班级和学生入手

### 2.2. URL 中 <u>**不要出现**</u> 动词

设计的 URL 中 **不要出现** 类似 `add`、`list`、`delete`、`start`、`stop` 等任何动词，如果以前没有设计过 Restful 风格接口的话这点会很不适应。但是我感觉这是 Restful 风格显得特别工整的关键。尤其多人的项目，质量又把控不严的情况下，各种五花八门的动词都出现了。例如在同一工程下同时用 `delXXX` 和 `deleteXXX` 表示删除。

同时这种设计使接口更简短直接，能很明白的知道操作的是什么资源。

## 3. Restful 中的常用方法

**GET** : 无论是不是 Restful 风格，这个应该使用最多的方法，获取数据，可能为单个资源或数组

**POST** : 也是特别常用的一个，Restful 风格中它代表的是添加，也就是数据库中的 insert

**PUT** : 更新，这个在非Restful 风格中可能用的不多，意思是更新，数据库中的 update

**DELETE** : 字面意思，删除

**PATCH*** : 这个用的稍微少一些，部分更新，我自己用的比较少，大部分接口实现 put 就够了

由于 Restful 风格中不允许使用动词，这几个方法就代替了动词的作用，需要合理使用

{{< admonition type=note title="注意" >}}
上面提到的 `start`、`stop` 等方法在 Restful 风格中不能直接体现，一般用 `PUT` 方法修改资源的状态代替，例如 put {"status": "start"}
{{< /admonition >}}

## 4. Restful 设计细节

### 4.1. 资源名称的设计

资源肯定都是名词，这都不用细说，另外还有一些官方推荐或我自己的一些建议：

1. 资源（包括url）全部小写
2. 资源都为复数，例如上面的班级，要设计成 classes，学生为 students
3. 如果资源是多个词，可用减号连接。例如 `书包` 可以写成 school-bags

### 4.2. 资源操作设计

其实就是如何使用上面的几种方法，这些都有既定的模板

#### 4.2.1. 获取资源列表

获取全部资源或者获取某种条件下的部分资源，条件可以通过参数传入
例如获取全部班级信息：

`GET` {base_url}/classes?name=一班

#### 4.2.2. 获取单个资源

通过 ID 获取资源，这里一般把 ID 设计到url路径中，例如：

`GET` {base_url}/classes/{class_id}

#### 4.2.3. 提交资源

提交（添加）资源，使用 `POST` 方法，参数一般为资源的json，例如:

`POST` {base_url}/classes

content可以为class的json

#### 4.2.4. 删除资源

同上，以班级为例
`DELETE` {base_url}/classes/{class_id}

#### 4.2.5. 更新资源

`PUT` 或 `PATCH` {base_url}/classes/{class_id}

参数与POST类似

#### 4.2.6. 资源操作模板

稍微总结一下，大部分的接口都可以套用模板

资源操作模板：



| 操作     | 方法         | url设计                            |
| -------- | ------------ | ---------------------------------- |
| 添加     | POST         | {base_url}/resources               |
| 获取全部 | GET          | {base_url}/resources               |
| 获取单个 | GET          | {base_url}/resources/{resource_id} |
| 删除     | DELETE       | {base_url}/resources/{resource_id} |
| 更新     | PUT 或 PATCH | {base_url}/resources/{resource_id} |

### 4.3. 资源分层设计

在上面的例子中，班级和学生是一对多的关系，可以认为有层级关系，如果想查询某个班里学生的情况，可以在班级后接学生的资源，例如：

`GET` {base_url}/classes/{class_id}/students

同时 students 资源也适用于上面的资源操作模板

{{< admonition type=tip title="tip" >}}
这里为了说明分层的设计，认为学生只依赖与班级存在
{{< /admonition >}}

### 4.4. 验证设计

有时接口中会出现验证的需求，例如传 token等

这里我一般是把验证信息放在请求的 header 里面，在 header 中添加

```html
Authorization: <type> <credentials>
```

例如 `Authorization: Basic YWRtaW46YWRtaW4=`

### 4.5. 状态设计

在 Restful 风格中，http的状态编码特别重要，这些状态码存在 header 中，可在数据处理前先验证状态，
这里我一般使用HTTP的默认状态码，我感觉一般情况下足够用了，详见 [status code](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status)

### 4.6. 返回设计

#### 4.6.1. GET 单个资源返回

我的建议是不要在外面套任何东西，请求什么资源就返回什么资源。
如果请求错误的话有 http status code 的错误代码，内容中可以加上错误信息。正确的话返回一个表示正确的代码有啥用？？在外面套一层没啥用的东西在我看来就是浪费解析资源。

#### 4.6.2. GET 列表

这里我的习惯是列表全部分页处理，分页参数可以依据需求设计，放在 URL 中，例如 `resources?pageSize=10&pageNum=10`。返回中包括分页的信息及数组，这里数组外面会包一层分页信息

#### 4.6.3. POST/PUT/PATCH 返回

我的建议是返回插入/修改后的资源，结构同单个资源返回

#### 4.6.4. DELETE 返回

这里可以什么都不返回，使用204。或返回内容，使用200。


### 4.7. 返回错误设计

接口中肯定要处理错误请求等，这时一定要返回的是上面提到的 http status code，错误内容可选返回或不返回，看需求处理

#### 4.7.1. 另外一些错误情况

1. get单个信息如果id没有找到一般返回 `404`，表示没有资源
2. 如果列表查询中，没有返回，可以返回 204，或灵活处理
3. 删除中遇到找不到要删除ID的情况，这时如果要删除的东西比较重要，可以返回 4xx

## 5. 总结

Restful 风格并不是一种强制的标准或规范，如何实现可以根据情况自行调整。
我感觉使用 Restful 风格能带来几点好处：

1. url 更加精简漂亮
2. 设计者和使用者一眼就能明白接口的作用
3. 类似模板的设计流程可以加快设计速度
4. 很多插件或工具对符合 Restful 风格的接口很友好，可以少些很多处理解析逻辑，很大程度上减少了代码量

## 6. 附录 restfulapi.net 推荐返回值



| 方法   | 集合(eg. /users)        | 单个项目 (eg. /users/123)              |
| ------ | ----------------------- | -------------------------------------- |
| POST   | 201                     | 405(Method Not Allowed)                |
| GET    | 200(OK)                 | 200(OK)/404(Not Found)                 |
| PUT    | 405(Method Not Allowed) | 200(OK)/404(Not Found)                 |
| PATCH  | 405(Method Not Allowed) | 200(OK)/404(Not Found)                 |
| DELETE | 405(Method Not Allowed) | 200(OK)/204(No Content)/404(Not Found) |


