## JSP 和 servlet 有什么区别？

JSP 是 servlet 技术的扩展，本质上就是 servlet 的简易方式。servlet 和 JSP 最主要的不同点在于，servlet 的应用逻辑是在 Java 文件中，并且完全从表示层中的 html 里分离开来，而 JSP 的情况是 Java 和 html 可以组合成一个扩展名为 JSP 的文件。JSP 侧重于视图，servlet 主要用于控制逻辑。

## JSP 有哪些内置对象？作用分别是什么？

JSP 有 9 大内置对象：

request：封装客户端的请求，其中包含来自 get 或 post 请求的参数；

response：封装服务器对客户端的响应；

pageContext：通过该对象可以获取其他对象；

session：封装用户会话的对象；

application：封装服务器运行环境的对象；

out：输出服务器响应的输出流对象；

config：Web 应用的配置对象；

page：JSP 页面本身（相当于 Java 程序中的 this）；

exception：封装页面抛出异常的对象。

## 说一下 JSP 的 4 种作用域？

page：代表与一个页面相关的对象和属性。

request：代表与客户端发出的一个请求相关的对象和属性。一个请求可能跨越多个页面，涉及多个 Web 组件；需要在页面显示的临时数据可以置于此作用域。

session：代表与某个用户与服务器建立的一次会话相关的对象和属性。跟某个用户相关的数据应该放在用户自己的 session 中。

application：代表与整个 Web 应用程序相关的对象和属性，它实质上是跨越整个 Web 应用程序，包括多个页面、请求和会话的一个全局作用域。

## 说一下 session 的工作原理？

session 的工作原理是客户端登录完成之后，服务器会创建对应的 session，session 创建完之后，会把 session 的 id 发送给客户端，客户端再存储到浏览器中。这样客户端每次访问服务器时，都会带着 sessionid，服务器拿到 sessionid 之后，在内存找到与之对应的 session 这样就可以正常工作了。

## 如果客户端禁止 cookie 能实现 session 还能用吗？

可以用，session 只是依赖 cookie 存储 sessionid，如果 cookie 被禁用了，可以使用 url 中添加 sessionid 的方式保证 session 能正常使用。

## Java Web 如何避免 SQL 注入？

使用预处理 PreparedStatement。

使用正则表达式过滤掉字符中的特殊字符。

## 请说一下表达式语言（EL）的隐式对象以及该对象的作用

EL的隐式对象包括：

- pageContext、initParam（访问上下文参数）

- param（访问请求参数）
- paramValues、header（访问请求头）
- headerValues、cookie（访问cookie）
- applicationScope（访问application作用域）

- sessionScope（访问session作用域）

- requestScope（访问request作用域）

- pageScope（访问page作用域）。

## 请谈一谈JSP有哪些内置对象？以及这些对象的作用分别是什么？

JSP有9个内置对象： 

- request：封装客户端的请求，其中包含来自GET或POST请求的参数； 

- response：封装服务器对客户端的响应；
- pageContext：通过该对象可以获取其他对象； 
- session：封装用户会话的对象；
- application：封装服务器运行环境的对象； 
- out：输出服务器响应的输出流对象； 
- config：Web应用的配置对象；
- page：JSP页面本身（相当于Java程序中的this）；
- exception：封装页面抛出异常的对象。

## 请简要说明一下JSP和Servlet有哪些相同点和不同点？另外他们之间的联系又是什么呢？

JSP 是Servlet技术的扩展，本质上是Servlet的简易方式，更强调应用的外表表达。JSP编译后是”类servlet”。Servlet和JSP最主要的不同点在于，Servlet的应用逻辑是在Java文件中，并且完全从表示层中的HTML里分离开来。而JSP的情况是Java和HTML可以组合成一个扩展名为.jsp的文件。JSP侧重于视图，Servlet主要用于控制逻辑

## 请谈谈你对Javaweb开发中的监听器的理解？

Java Web开发中的监听器（listener）就是application、session、request三个对象创建、销毁或者往其中添加修改删除属性时自动执行代码的功能组件，

如下所示： 

①ServletContextListener：对Servlet上下文的创建和销毁进行监听。

②ServletContextAttributeListener：监听Servlet上下文属性的添加、删除和替换。 

③HttpSessionListener：对Session的创建和销毁进行监听。

session的销毁有两种情况：

1). session超时（可以在web.xml中通过/标签配置超时时间）；

2). 通过调用session对象的invalidate()方法使session失效。 

④HttpSessionAttributeListener：对Session对象中属性的添加、删除和替换进行监听。 

⑤ServletRequestListener：对请求对象的初始化和销毁进行监听。 

⑥ServletRequestAttributeListener：对请求对象属性的添加、删除和替换进行监听。

## 请回答一下servlet的生命周期是什么。servlet是否为单例以及原因是什么？

Servlet 生命周期可被定义为从创建直到毁灭的整个过程。以下是 Servlet 遵循的过程：

Servlet 通过调用 init () 方法进行初始化。

Servlet 调用 service() 方法来处理客户端的请求。

Servlet 通过调用 destroy() 方法终止（结束）。

最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的。

Servlet单实例，减少了产生servlet的开销；

## 请简要说明一下forward与redirect区别，并且说一下你知道的状态码都有哪些？以及redirect的状态码又是多少？

1.从地址栏显示来说

forward是服务器请求资源,服务器直接访问目标地址的URL,把那个URL的响应内容读取过来,然后把这些内容再发给浏览器.浏览器根本不知道服务器发送的内容从哪里来的,所以它的地址栏还是原来的地址.

redirect是服务端根据逻辑,发送一个状态码,告诉浏览器重新去请求那个地址.所以地址栏显示的是新的URL.

2.从数据共享来说

forward:转发页面和转发到的页面可以共享request里面的数据.

redirect:不能共享数据.

3.从运用地方来说

forward:一般用于用户登陆的时候,根据角色转发到相应的模块.

redirect:一般用于用户注销登陆时返回主页面和跳转到其它的网站等.

4.从效率来说

forward:高.

redirect:低.

redirect的状态码是302

## 请谈一谈，get和post的区别？

1. 携带请求参数的方式

- GET: 通过请求行携带参数, 参数会显示在地址栏
- POST: 通过请求体来携带参数, 参数不会显示在地址栏 

2. 服务器端处理请求的方法

- GET: 会调用 Servlet 的 doGet()来处理请求
- POST: 会调用 Servlet 的 doPost()来处理请求 

3. 数据大小与安全性
- GET: 大小有限制(小于 2k), 不安全 POST: 大小没有限制, 安全

## 请你说说，cookie 和 session 的区别？

1、cookie数据存放在客户的浏览器上，session数据放在服务器上。

2、cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗

考虑到安全应当使用session。

3、session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能，考虑到减轻服务器性能方面，应当使用COOKIE。

4、单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。

## jsp 有哪些动作?作用分别是什么?

JSP 共有以下 6 种基本动作 

- jsp:include:在页面被请求的时候引入一个文件。 

- jsp:forward:把请求转到一个新的页面。 

- jsp:useBean:寻找或者实例化一个 JavaBean。

- jsp:setProperty:设置 JavaBean 的属性。

- jsp:getProperty:输出某个 JavaBean 的属性。 

- jsp:plugin:根据浏览器类型为 Java 插件生成 OBJECT 或 EMBED 标记

## MVC 的各个部分都有那些技术来实现?如何实现?

MVC 是 Model-View-Controller 的简写。

Model 代表的是应用的业务逻辑(通过 JavaBean，EJB 组件实现)，

View 是应用的表示面(由 JSP 页面产生)，

Controller 是提供应用的处理过程控制(一般是一个 Servlet)， 

通过这种设计模型把应用逻辑，处理过程和显示逻辑分成不同的组件实现。这
些组件可以进行交互和重用。

