# Web容器

本文主要介绍Web容器相关的知识, 包括Web容器的定义, 种类和比较, 以及本地安装.

## 什么是Web容器？

Web容器是什么? Web容器和Web服务器(Web Server)有什么区别？ Web容器的作用有哪些？

Web容器是Web服务器的一个组件, 主要负责与Java servlet交互, 管理servlet的生命周期, 将URL映射到特定的servlet进行处理等功能.

Web服务器, 也就是常称的HTTP服务器, 通过HTTP协议为客户端提供Web页面的服务端, 本身具有存储, 处理请求等功能. 其中提供的页面最常见的是HTML文档, 除了文本内容之外, 还可以包括图像, 样式表和脚本等类型文件. 目前世界上主流的Web服务器占比为: Apache: 44.6%, Nginx: 40.0%, IIS: 9.5%.(注数据来源:[W3Techs](https://w3techs.com/technologies/overview/web_server/all)).

用一个简单的比喻来形容两者的关系, Web服务器好比一辆大卡车给别人运货, 如果你运送的是文本(HTML), 图片(IMAGE), 样式表(CSS)这些固体货物的话, 卡车已经可以装货运输. 但是如果需要运输类似汽油(Java Servlet)这类的液体货物, 那就需要使用一个桶(Web容器)装好之后, 再运输. 而Web容器就是这个桶, 主要处理这些液体货物(Java Servlet), 所以它也有一个别称: Servlet容器.

Web容器实现了**J2EE**中关于网络组件(Web component)[相关规范](https://en.wikipedia.org/wiki/Java_servlet), 该规范为定义了Web组件在运行时环境的相关信息和配置, 包括安全性, 并发性, 生命周期管理, 事务, 部署和其他服务. 就好比`Java Servlet`的保姆, 对其一生进行负责.

HTTP请求的简单处理流程: 当一个请求发送到达服务器, Web容器开启一个线程进行处理, 线程根据URL信息映射到容器内部中特定的Servlet进行处理, 处理结束之后将结果返回结束线程.

## Web容器的种类

`开源Web容器`:Apache Tomcat, Apache Geronimo, Jetty, GlassFish, Jaminid, Enhydra, Payara, Winstone, Tiny Java Web Server, WildFly, Virgo, JBoss.

`商业Web容器`:JBoss Enterprise Application Platform, WebSphere Application Server, WebLogic Application Server, iPlanet Web Server, JRun, Orion Application Server, Resin Pro, ServletExec, SAP NetWeaver, tc Server.

这里主要介绍其中比较有代表性的Tomcat, JBoss, WebSphere, WebLogic.

### Web容器比较

|   Web容器    | Tomcat                                                                                                                       | JBoss                                                                             | WebSphere                                                                                                                 | WebLogic                                                                                                                                                                     |
|------------|:-----------------------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------|:--------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 开发者        | Apache                                                                                                                       | JBoss/Redhat                                                                      | IBM                                                                                                                       | BEA/Oracle                                                                                                                                                                   |
| 开源/License | YES/Apache License                                                                                                           | YES/LGPL License                                                                  | NO                                                                                                                        | NO                                                                                                                                                                           |
| 费用         | Free                                                                                                                         | Free                                                                              | Very expensive                                                                                                            | expensive                                                                                                                                                                    |
| 文档完备性      | 中                                                                                                                            | 中                                                                                 | 高                                                                                                                         | 高                                                                                                                                                                            |
| 社区活跃度      | 高                                                                                                                            | 中                                                                                 | 中                                                                                                                         | 中                                                                                                                                                                            |
| 实现功能       | JSP,Servlet                                                                                                                  | EJB                                                                               | Servlet,EJB,JSB,JMS,JDBC,XML,WML(ALL)                                                                                     | Servlet,EJB,JSB,JMS,JDBC,XML,WML(ALL)                                                                                                                                        |
| 服务和技术支持    | 源码/无                                                                                                                         | 源码/无                                                                              | 技术文档/技术人员(快速/收费)                                                                                                          | 技术文档/技术人员(快速/收费)                                                                                                                                                             |
| 安全性        | 低(未验证)                                                                                                                       | 低(未验证)                                                                            | 高(已验证)                                                                                                                    | 高(已验证)                                                                                                                                                                       |
| 应用范围       | 轻量级,中小型系统                                                                                                                    | 需要EJB服务的中小型系统                                                                     | 重要,大型的系统                                                                                                                  | 重要,大型的系统                                                                                                                                                                     |
| 评价         | 由于开发团队有了Sun的参与和支持,最新的Servlet和JSP规范总是能在Tomcat 中得到体现. 因为Tomcat 技术先进,性能稳定,而且免费,因而深受Java爱好者的喜爱并得到了部分软件开发商的认可,成为目前比较流行的Web 应用服务器. | 支持EJB 1.1,EJB 2.0和EJB3.0的规范.但JBoss核心服务不包括支持servlet/JSP的WEB容器,一般与Tomcat或Jetty绑定使用. | WebSphere是 IBM 的集成软件平台.它包含了编写,运行和监视全天候的工业强度的随需应变 Web 应用程序和跨平台,跨产品解决方案所需要的整个中间件基础设施,如服务器,服务和工具.WebSphere 提供了可靠,灵活和健壮的集成软件. | WebLogic是美国bea公司出品的一个application server确切的说是一个基于j2ee架构的中间件.BEA WebLogic是用于开发,集成,部署和管理大型分布式Web应用,网络应用和数据库应用的Java应用服务器.将Java的动态功能和Java Enterprise标准的安全性引入大型网络应用的开发,集成,部署和管理之中. |

## 相关名词解释

Apache HTTP Server: 由`Apache`开发维护的, 免费的开源跨平台Web服务器软件, 根据Apache License 2.0的许可进行发布.

Nginx: 知名的高性能Web服务器, 免费且以类BSD的license进行发布源码, 常被用作反向代理服务器, 负载均衡器, 邮件代理和HTTP缓存.

IIS: `Internet Information Services`, 微软公司出版, Windows服务器的标配. 提供HTTP, HTTP/2, HTTPS, FTP, FTPS, SMTP和NNTP等功能.

HTTP: `HyperText Transfer Protocol`, 超文本传输协议, 应用层协议, 万维网通信的基础.

Apache: `Apache Software Foundation`(简称ASF), Apache软件基金会, 一个支持开源软件项目的非盈利性组织, 维护和支持了非常多优秀的开源项目, 如Maven, Hadoop, POI, Struts, Ant等等.

JBoss/Redhat: JBoss开始由JBoss公司开发和维护, 在2006年的时候JBoss公司被Redhat收购. 2014年11月20日软件更名为`WildFly`.

IBM: `International Business Machines Corporation`, 全球最大的信息技术和业务解决方案公司, 拥有全球雇员30多万人, 业务遍及160多个国家和地区.

BEA/Oracle: `BEA Systems, Inc`, 著名的Java中间件公司, 曾经在Java中间件中的市场份额一度超过IBM, 后在2008年4月29日被Oracle收购.

## 实际操作

这里实际操作, 以开源tomcat进行本地安装测试.

### 下载安装

+ 下载地址: `https://tomcat.apache.org/`, 注意配置管理员账号和密码.
+ 使用maven构建: `mvn archetype:generate` 获取远程原型列表, 输入: `maven-archetype-webapp`进行过滤, 选择`apache`的官方原型, 输入配置信息.
+ 进入项目文件夹, 进行打包: `mvn package`.
+ 登录`http//localhost:8080/`进行部署.

## 参考文档

[容器比较](http://www.cnblogs.com/kaleidoscope/p/9668646.html)

[wikipedia](https://en.wikipedia.org/wiki/Web_container)
