RESTful 风格 API 设计实践指南
==============================
围绕以“资源”为中心的抽象
------------------------------

## 起源

REST这个词，是Roy Thomas Fielding在他2000年的博士论文中提出的。

> 本文研究计算机科学两大前沿----软件和网络----的交叉点。长期以来，软件研究主要关注软件设计的分类、设计方法的演化，很少客观地评估不同的设计选择对系统行为的影响。而相反地，网络研究主要关注系统之间通信行为的细节、如何改进特定通信机制的表现，常常忽视了一个事实，那就是改变应用程序的互动风格比改变互动协议，对整体表现有更大的影响。我这篇文章的写作目的，就是想在符合架构原理的前提下，理解和评估以网络为基础的应用软件的架构设计，得到一个功能强、性能好、适宜通信的架构。<br>

Fielding将他对互联网软件的架构原则，定名为REST，即Representational State Transfer的缩写。我对这个词组的翻译是"表现层状态转化"。

如果一个架构符合REST原则，就称它为RESTful架构。要理解RESTful架构，最好的方法就是去理解Representational State Transfer这个词组到底是什么意思，它的每一个词代表了什么涵义。

## 基本规则

HTTP协议不是专门用来做RPC的协议框架。基于资源的抽象方法是非常完备的，几乎可以描述绝大部分的世界现象。

### 资源定位 - URL(统一资源定位符)

在RESTful架构中，每个网址代表一种资源（resource），所以网址中不能有动词，只能有名词，而且所用的名词往往与数据库的表格名对应。

我们约定，使用资源的类名称时，指代它的一种聚合结构。

因为依据其他同行在实践复数类名称作为聚合的路径时，经常遇到英文复数名词的特殊变化，为降低心智负担，统一词汇覆盖为上述约定。

举例来说，有一个API提供动物园（zoo）的信息，还包括各种动物和雇员的信息，则它的路径应该设计成下面这样。

 > * https://api.example.com/v1/zoo
 > * https://api.example.com/v1/animal
 > * https://api.example.com/v1/employee

主要结构包括:
* 根命名空间（可选） - 与其他页面路径隔离，例如 ```/api/...```，建议使用，可以提高研发和部署时的便利。
* 接口版本（可选）- 可以标识接口版本，或并存可兼容版本 ```/api/v1/...```
* 资源命名空间（可选） - 为资源命名消歧义，或预防今后产生歧义，以credit为例```/api/v1/credit/protocol/...```，其中credit指信用业务，所以限定credit/protocol为“信用业务的协议资源”。
* 资源路径 - 资源定义：聚合对象```.../api/v1/credit/protocol```；实例对象```.../api/v1/credit/protocol/1```

常见错误：

**包含动词/动宾短语** 当开发者在URL中使用动词或动宾短语时，其本质是把http协议作为rpc来使用，这是必须被禁止的。以创建一个信用协议为例，```POST /api/v1/credit/protocol/create```应该为```POST /api/v1/credit/protocol```

### 动作 - method(HTTP Method)

| Verb    | Method   | Idempotent | Description                                   |
| ------- | -------- | ---------- | --------------------------------------------- |
| GET     | Select   | Yes        | 从服务器取出资源（一项或多项）                   |
| POST    | Create   | No         | 在服务器新建一个资源                            |
| PUT     | Update   | Yes        | 在服务器更新资源（客户端提供改变后的完整资源）    |
| DELETE  | Delete   | Yes        | 从服务器删除资源                               |
| PATCH   | Update   | No         | 按照一组操作序列完成对指定资源的增量更新         |

一些举例：

> - GET /zoo - 列出所有动物园
> - POST /zoo - 新建一个动物园
> - GET /zoo/{zooId} - 获取某个指定动物园的信息
> - PUT /zoo/{zooId} - 更新某个指定动物园的信息（提供该动物园的全部信息）
> - DELETE /zoo/{zooId} - 删除某个动物园
> - GET /zoo/{zooId}/animal - 列出某个指定动物园的所有动物
> - DELETE /zoo/{zooId}/animal/{animalId} - 删除某个指定动物园的指定动物
> - PATCH /zoo/{zooId} - 更新某个指定动物园的信息（提供该动物园的部分信息）

### 过滤信息

指形如```?currentPage=1&perPage=10```结尾的参数，在部分语境里，我们也称之为：过滤参数、Query参数、QueryString、Query等。

过滤信息主要用于修饰请求，这些参数通常不作为资源的组成要素，但有包含特定作用。例如，如果请求一个组织机构的查询聚合时。想按照关键字返回包含关键字的查询结果，则：```.../organization```->```.../organization?keyword=银行```，来修饰获取组织机构聚合的查询行为。但可以想象到在Response里，并不会包含keyword这个要素。

所以，基于该规则，可以渐进提升服务端的查询功能。

### 状态码

服务器向用户返回的状态码和提示信息，常见的有以下一些（方括号中是该状态码对应的HTTP动词）。

> - 200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
> - 201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
> - 202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
> - 204 NO CONTENT - [DELETE]：用户删除数据成功。
> - 400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
> - 401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
> - 403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
> - 404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
> - 406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
> - 410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
> - 422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
> - 500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功。

状态码索引

```
100 "continue"
101 "switching protocols"
102 "processing"
200 "ok"
201 "created"
202 "accepted"
203 "non-authoritative information"
204 "no content"
205 "reset content"
206 "partial content"
207 "multi-status"
208 "already reported"
226 "im used"
300 "multiple choices"
301 "moved permanently"
302 "found"
303 "see other"
304 "not modified"
305 "use proxy"
307 "temporary redirect"
308 "permanent redirect"
400 "bad request"
401 "unauthorized"
402 "payment required"
403 "forbidden"
404 "not found"
405 "method not allowed"
406 "not acceptable"
407 "proxy authentication required"
408 "request timeout"
409 "conflict"
410 "gone"
411 "length required"
412 "precondition failed"
413 "payload too large"
414 "uri too long"
415 "unsupported media type"
416 "range not satisfiable"
417 "expectation failed"
418 "I'm a teapot"
422 "unprocessable entity"
423 "locked"
424 "failed dependency"
426 "upgrade required"
428 "precondition required"
429 "too many requests"
431 "request header fields too large"
500 "internal server error"
501 "not implemented"
502 "bad gateway"
503 "service unavailable"
504 "gateway timeout"
505 "http version not supported"
506 "variant also negotiates"
507 "insufficient storage"
508 "loop detected"
510 "not extended"
511 "network authentication required"
```

对于研发人员来说，如果当前你不知道这算什么错误，按照状态码的要求，置statusCode就可以，随着实现过程中，规格逐渐清晰，再进一步补充错误处理。

### 错误处理 - （4xx, 5xx）

任何请求，当发生错误时，都可以返回错误对象资源作为响应结果。以一个400的错误为例，它的响应Payload可以是:

```json
{
    "message": "Invalid API key"
}
```

实际研发实践中，statusCode是客户端作为首要的异常处理依据，当需求上需要进一步区分时，才将进一步处理错误资源对象的信息。所以这也揭示了，服务端在实现功能时，可以渐进的丰富改内容，甚至可以长期保持原始的状态码含义。

### 载荷&模型

对于POST、PUT这样的有载荷方法，需要确保请求载荷与响应载荷对称。确保来去载荷结构一致，可以使接口的约定粒度从每个API提高到每种资源。减少对接时的沟通成本。

**强一致** 结构完整，不缺少属性，不多属性，无数据项用null代替，例如创建时不知道id的值，但可以写为```{ id: null }```。

**弱一致** 不缺少关键属性

### 实践 - CreditMetric - 信用指标

以“强一致”载荷要求为例：

模型,
```json
{
    "id": 0,
    "code": "",
    "name": "",
    "status": "",
    "aspect": "",
    "dataType": "",
    "sourceType": "",
    "subjectType": "",
    "description": "",
    "condition": "",
    "createdAt": "2020-02-04T09:53:28.584Z",
    "updatedAt": "2020-02-04T09:53:28.584Z",
    "createdBy": <User Model>,
    "updatedBy": <User Model>
}
```

URL和Method，

```
GET /credit/metric/{metricId} 查询某个指标

PUT /credit/metric/{metricId} 修改某个指标

GET /credit/metric 查询指标列表

POST /credit/metric 创建指标
```

创建一个“信用指标”的请求与响应例子：

Request,
```
POST /credit-server/credit/metric HTTP/1.1
Host: 192.168.1.179:3000
Connection: keep-alive
Content-Length: 131
Pragma: no-cache
Cache-Control: no-cache
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.122 Safari/537.36
Content-Type: application/json;charset=UTF-8
Origin: http://192.168.1.179:3000
Referer: http://192.168.1.179:3000/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: imgCodeKey=vrj71gb9dq; Authorization=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWRtaW5AY2NjY2l0LmNvbS5jbiIsImVudiI6ImRldiIsImlzcyI6ImNjY2NpdCIsInNsdCI6IklsQSJ9.cxI9M-Qd1fbOAabESG62M8qscCmleuQZB30rvTemZDw; Principal=admin@ccccit.com.cn


{
    "id": null,
    "code": "fdsfsfs",
    "name": "fdsfsdf",
    "status": null,
    "aspect": "0",
    "dataType": "0",
    "sourceType": "2",
    "subjectRole": "2",
    "conditions": "fdsfsfsf",
    "description": "fsdfsdfsfs",
    "createdAt": null,
    "updatedAt": null,
    "createdBy": null,
    "updatedBy": null
}
```

Response,
```
HTTP/1.1 200 OK
content-type: application/json;charset=UTF-8
transfer-encoding: chunked
date: Sat, 29 Feb 2020 05:13:22 GMT
connection: close


{
    "id": 20,
    "code": "fdsfsfs",
    "name": "fdsfsdf",
    "status": "1",
    "aspect": "0",
    "dataType": "0",
    "sourceType": "2",
    "subjectRole": "2",
    "description": "fsdfsdfsfs",
    "conditions": "fdsfsfsf",
    "createdAt": "2020-02-29T13:13:23.000Z",
    "updatedAt": "2020-02-29T13:13:23.000Z",
    "createdBy": {
        id: "1",
        name: "admin@ccccit.com.cn"
    },
    "updatedBy": {
        id: "1",
        name: "admin@ccccit.com.cn"
    }
}
```

## 附录

* https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html
* http://www.ruanyifeng.com/blog/2014/05/restful_api.html
* https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm