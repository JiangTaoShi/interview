#### 1、介绍一下springMVC

springmvc是一个视图层框架，通过MVC模型让我们很方便的接收和处理请求和响应。我给你说说他里边的几个核心组件吧

它的核心控制器是DispatcherServlet，他的作用是接收用户请求，然后给用户反馈结果。它的作用相当于一个转发器或中央处理器，控制整个流程的执行，对各个组件进行统一调度，以降低组件之间的耦合性，有利于组件之间的拓展

接着就是处理器映射器（HandlerMapping）：他的作用是根据请求的URL路径，通过注解或者XML配置，寻找匹配的处理器信息

还有就是处理器适配器（HandlerAdapter）：他的作用是根据映射器处理器找到的处理器信息，按照特定执行链路规则执行相关的处理器，返回ModelAndView

最后是视图解析器（ViewResolver）：他就是进行解析操作，通过ModelAndView对象中的View信息将逻辑视图名解析成真正的视图View返回给用户

接下来我给你说下springmvc的执行流程吧

#### 2、springMVC的执行流程

（1）用户发送请求至前端控制器 DispatcherServlet； 

（2） DispatcherServlet 收到请求后，调用 HandlerMapping 处理器映射器，请求获取 Handle； 

（3）处理器映射器根据请求 url 找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给 DispatcherServlet； 

（4）DispatcherServlet 调用 HandlerAdapter 处理器适配器； 

（5）HandlerAdapter 经过适配调用 具体处理器(Handler，也叫后端控制器)； 

（6）Handler 执行完成返回 ModelAndView； 

（7） HandlerAdapter 将 Handler 执 行 结 果 ModelAndView 返 回 给DispatcherServlet； 

（8）DispatcherServlet 将 ModelAndView 传给 ViewResolver 视图解析器进行解析； 

（9）ViewResolver 解析后返回具体 View； 

（10）DispatcherServlet 对 View 进行渲染视图（即将模型数据填充至视图中） 

（11）DispatcherServlet 响应用户。



#### 3、springMVC接收前台参数的几种方式

- 1、如果传递参数的时候，通过ur1拼接的方式，直接拿对象接收即可，或者string、 int

- 2、如果传递参数的时候，传到后台的是js对象，那么必须使用对象接收，并且加@requestBody，使用requestBody之后，传递的参数至少要有一个，并且所有传的参数都要在后台对象里存在

- 3、get的请求方式，所有的参数接收都使用普通对象或者string、int

- 4、在用form表单提交的时候，所有的参数接收都使用普通对象或者string、int



#### 4、springMVC中的常用注解

@RequestMapping：指定类或者方法的请求路径，可以使用method字段指定请求方式

@GetMapping、@PostMapping：规定了请求方式的方法的请求路径

@RequestParam：接收单一参数的

@PathVariable：用于从路径中接收参数的

@CookieValue：用于从cookie中接收参数的

@RequestBody：用于接收js对象的，将js对象转换为Java对象

@ResponseBody：返回json格式数据

@RestController：用在类上，等于@Controller+@ResourceBody两个注解的和，一般在前后端分离的项目中只写接口时经常使用，标明整个类都返回json格式的数据

等等



#### 5、spring如何整合springMVC

简单的说 springMVC在ssm中整合 就是 在 web.xml 里边配置springMVC的核心控制器:DispatcherServlet; 它就是对指定后缀进行拦截;然后在springMVC.xml里边配置扫描器，可以扫描到带@controller注解的这些类，现在用springMVC都是基与注解式开发， 像@service，@Repository @Requestmapping，@responsebody 啦这些注解标签 等等 都是开发时用的，每个注解标签都有自己的作用;它还配置一个视图解析器，主要就是对处理之后的跳转进行统一配置，有页面的路径前缀和文件后缀 ，如果有上传相关的设置，还需要配置上multpart的一些配置，比如单个文件最大的大小，以及最大请求的大小，大致就是这些

















