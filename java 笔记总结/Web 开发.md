浏览器内核是去渲染前端代码的东西，为了让网站在每个浏览器显示的效果一样，于是产生了 w3c 万维网。

HTML 负责结构
CSS 负责表现
javascript 负责网页行为（交互效果）
大多数公司基本都是前后端分离，通过 HTTP 接口通信，通常传 JSON


Springboot 到底是什么： 一个快速帮我开发 java后端的网站框架
没有 SB ： 自己要去配置 tomcat 啦，servlet 啦等等等
Springboot = Spring + 自动配置+ 内置服务器


Spring 容器 = Spring 自己维护的一个超大对象仓库

注意注意：
java 本身的类确实是说明书，但是需要注意 Spring 管理的不是类，而是对象

new 了之后是真正创建了一个对象

Spring 管理的是这个产生出来的对象


Springboot 在整个网站里面干什么？
接受请求 - 》 找到对应 conttroller - 调用 Service 查数据库 -返回 JSON
Springboot 里面的角色：
Controller ： 接收前端请求
Service ：真正干活
Mapper ： Mapper 只有一个工作：访问数据库 mapper 其实就是 java 和MYSQL 之间的翻译
MYSQL：真正保存数据

 Springboot 为什么好？
 


能做什么：后台管理类系统比如 
CRM 管客户系统
ERP 管 公司内部资源系统

web 前端：
![[Pasted image 20260711175041.png]]
html JS





maven： java 项目的包管理+项目管理 工具

三个角色：
依赖管理 - 自动下载和管理第三方 jar 包
项目管理 统一项目目录，配置和脚本
构建工具 编译测试，打包，发布项目

MVC : java 经典设计思想 - 一个程序分成三层，每层只负责自己的事情。
M -model V-View  C-controller




sevver socket 负责接受网络连接， HTTP 负责规定消息格式，Tomcat 负责解析 http 并调用你的 spring 代码

整个过程：
浏览器 - TCP 连接-server socket - 收到字节流 - tomcat 去解析 HTTP，然后 Spring MVC 找到 controller 做一系列事情，然后返回 HTTP response

servlet 程序 = 按照 servlet 规范写的，专门处理 web 请求的 java 程序


dispaterche 意思是调度器分发器 - 底层的 dispatcher servlet 就是要把这个任务交给谁然后分发出去，那么其实也就是分发给 conteoller

Spring ：
控制反转： 其实本质上是为了拒绝耦合，为什么呢，比如你制造的一个类，然后呢，你这个东西可能要被 new 很多次，那玩意你要改，你就得多个地方一起改。
inversion of  control 简称依赖注入，IOC。 

依赖注入：其实就是容器给应用程序提供所依赖的资源
bean 对象就是 IOC 容器中创建管理的对象  
bean 注入不能存在多个相同类型的 bean：
可解方案：
@primary:
设置 bean 优先级 （bean 里（
@qualifier 指定 bena 名字（定义里）

为什么需要连接池？ 创建数据库连接本身有成本，非常浪费
Mybatis 数据库连接池
一个容器，负责分配、管理数据库连接
允许 app 重复使用一个现有的数据库连接
释放空间时间超过最大空闲时间的连接，避免连接遗漏

SB 默认使用 hikari 这个连接池
德鲁伊四阿里巴巴开源，非常强

Druid
优势是资源服用，提升系统响应速度

lombok 是一个实用 java 类库
@getter setter
@tostring
@


AOP 面向切面编程：


