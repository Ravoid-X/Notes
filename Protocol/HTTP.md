## 概述
全称为“超文本传输协议”，属于应用层协议
<img src="../Pic/Protocol/http-example.jpg" style="width:500px;padding:10px;"/>

## URI
1. Uniform Resource Identifier（统一资源标志符），包括 URL 和 URI\
<img src="../Pic/Protocol/http-uri.jpg" style="width:300px;padding:10px;"/>

2. URL：Universal Resource Locator（统一资源定位符），即网址。在互联网中，URN 使用的非常少，几乎所有的 URI 都是 URL。
3. URI：Universal Resource Name（统一资源名称），只为资源命名而不指定如何定位资源。
### URL格式
在URL规定格式中，括号包括内容是非必要部分\
<img src="../Pic/Protocol/http-url.jpg" style="width:700px;padding:10px;"/>

1. scheme：协议。常用的协议有 http、https、ftp等，另外 scheme 也常常被称为 protocol。
2. username，passowrd：用户名和密码，有时候登陆时会显示，现在用到比较少
3. hostname：主机地址，该地址可以是域名也可以是IP地址。
4. port：端口，这个就是服务器设定的服务端口，例如 https:127.0.0.1:8080 的端口是 8080。URL 中没有端口信息是使用了默认端口，例如 http 协议的默认端口是 80，https 协议的默认端口是 443。
5. path：路径，指的就是网络资源在服务器中的指定地址
6. parameters：参数，用来指定访问某个资源时的附加信息
7. query：查询，用来查询某个类资源，如果又多个资源，则用 & 隔开
8. fragment：片段，是对资源描述的部分补充。
### URL encode
像 $/ ? : @$ 等字符，已经被URL当做特殊字符使用。如果 URL 中查询部分涉及到这些特殊字符，就可能导致该 URL 解析错误。针对这种情况，需对 URL 中的特殊字符进行转义，即为 URL encode。\
``https://www.w3schools.com/tags/ref_urlencode.ASP``

## HTTP 协议格式
可以分为两种，HTTP 请求报文格式 和 HTTP 响应报文格式。\
在浏览器访问 www.baidu.com，使用 F12 -> network 可查看报文\
<img src="../Pic/Protocol/http-message.jpg" style="width:700px;padding:10px;"/>

## 请求报文格式
<img src="../Pic/Protocol/http-message-request.jpg" style="width:700px;padding:10px;"/>

### 起始行
1. 版本：报文所使用的 HTTP 版本，格式如：HTTP/<major>.<minor>
2. 方法：\
（1）GET：从服务器获取一份文档\
（2）HEAD：只从服务器获取文档的首部，从而在不获取资源的情况下了解资源的情况\
（3）POST：向服务器发送需要处理的数据\
（4）PUT：将请求的主体部分存储在服务器上\
（5）TRACE：客户端发起一个请求，中间可能会经过网关、防火墙等等应用程序，从而导致原始 HTTP 请求被篡改。Trace 请求可以对报文进行追踪，当服务器收到最终请求报文时，会返回客户端一个 Trace 响应告知它所收到的最终请求报文\
（6）OPTIONS：请求服务器告知其支持哪些方法\
（7）DELETE：从服务器上删除一份文档\
3. 状态码：在每条响应报文的起始行中返回，后跟一个原因短语便于理解。可以通过三位数字代码对不同状态码进行分类，200 到 299 之间的状态码表示成功，300 到 399 之间的代码表示资源已经被移走了，400 到 499 之间的代码表示客户端的请求出错，500 到 599 之间的代码表示服务器出错\
### 首部
HTTP 首部字段向请求和响应报文中添加了一些附加信息，本质上来说，它们只是一些键值对的列表
#### 通用首部：既可以出现在请求报文中，也可以出现在响应报文中\
（1）Connection：允许客户端和服务器指定与请求或响应连接有关的选项\
（2）Date：提供日期和时间标志，说明报文是什么时间创建的\
（3）MIME-Version：给出了发送端使用的 MIME 版本\
（4）Transfer-Encoding：告知接收端为了保证报文的可靠传输，对报文采用了什么编码方式\
（5）Update：给出了发送端可能想要升级使用的新版本或协议\
（6）Via：显示了报文经过的中间节点（代理、网关）
#### 请求首部：
1. 常用的信息性首部\
（1）Client-IP：提供了运行客户端的机器的IP地址\
（2）From：提供了客户端用户的E-mail地址\
（3）Host：给出了接收请求的服务器的主机名和端口号\
（4）Referer：提供了包含当前请求URI的文档的URL\
（5）UA-OS：给出了运行在客户端机器上的操作系统名称及版本\
（6）User-Agent：将发起请求的应用程序名称告知服务器
2. Accept 首部为客户端提供了一种将其喜好和能力告知服务器的方式\
（1）Accept：告诉服务器能够发送哪些媒体类型）\
（2）Accept-Charset：告诉服务器能够发送哪些字符集\
（3）Accept-Encoding：告诉服务器能够发送哪些编码方式\
（4）Accept-Language：告诉服务器能够发送哪些语言
3. 条件请求首部\
（1）Expect：允许客户端列出某请求所要求的服务器行为\
（2）If-Match：如果实体标记与文档当前的实体标记相匹配，就获取这份文档\
（3）If-Modified-Since：除非在某个指定的日期之后资源被修改过，否则就限制这个请求\
（4）If-None-Match：如果提供的实体标记与当前文档的实体标记不相符，就获取文档\
（5）If-Range：允许对文档的某个范围进行条件请求\
（6）If-Unmodified-Since：除非在某个指定日期之后资源没有被修改过，否则就限制这个请求\
（7）Range：如果服务器支持范围请求，就请求资源的指定范围
4. 安全请求首部\
（1）Authorization：包含了客户端提供给服务器，以便对其自身进行认证的数据\
（2）Cookie：客户端用它向服务器传送一个令牌
5. 代理请求首部\
（1）Max-Forward：在通往源端服务器的路径上，将请求转发给其他代理或网关的最大次
数，往往与 TRACE 方法一同使用\
（2）Proxy-Authorization：与 Authorization 首部相同，但这个首部是在与代理进行认证时使用的\
（3）Proxy-Connection：与 Connection 首部相同，但这个首部是在与代理建立连接时使用的
#### 响应首部
提供更多有关响应的信息，比如谁在发送响应，响应者的功能，有助于客户端更好的处理请求
#### 实体首部
描述主体的长度和内容，或者资源自身
1. 信息型首部\
（1）Allow：列出了可以对此实体执行的请求方法\
（2）Location：告知客户端实体实际上位于何处，用于重定向
2. 内容首部：与实体内容有关的特定信息，说明了其类型、尺寸以及处理它所需的其他有用信息\
（1）Content-Base：解析主体中的相对 URL 时使用的基础 URL\
（2）Content-Encoding：对主体执行的任意编码方式\
（3）Content-Language：理解主体时最适宜使用的自然语言\
（4）Content-Length：主体的长度或尺寸\
（5）Content-Location：资源实际所处的位置\
（6）Content-MDS：主体的 MD5 校验和\
（7）Content-Range：在整个资源中此实体表示的字节范围\
（8）Content-Type：这个主体的对象类型
3. 缓存首部\
（1）Expires：实体不再有效，需要重新获取\
（2）Last-Modified：实体最后一次被修改的日期和时间
### 主体部分
HTTP 要传输的内容

## 响应报文格式
<img src="../Pic/Protocol/http-message-respond.jpg"  style="width:400px;padding:10px;"/>
