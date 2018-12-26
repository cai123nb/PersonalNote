# Web容器
Web容器(Web Container)相关的知识科普.

## 什么是Web容器？
Web容器是什么? Web容器和Web服务器(Web Server)有什么区别？ Web容器的作用有哪些？

Web容器是Web服务器的一个组件, 主要负责与Java servlet交互, 管理servlet的生命周期, 将URL映射到特定的servlet进行处理等. 

Web服务器的主要是提供存储,处理和向客户端提供Web页面的服务端. 客户端和服务器之间的通信使用超文本传输协议(HTTP)进行. 提供的页面最常见的是HTML文档, 除了文本内容之外, 还可能包括图像, 样式表和脚本. 目前世界上使用最多的Web服务器为: Apache: 44.6%, Nginx: 40.0%, IIS: 9.5.(注数据来源:[W3Techs](https://w3techs.com/technologies/overview/web_server/all)).

用一个简单的比喻来形容两者的关系, Web服务器好比一辆大卡车给别人运货, 如果你运送的是文本(HTML), 图片(IMAGE), 样式表(CSS)这些固体货物的话, 卡车已经可以完成运输. 但是如果需要运输类似汽油(Java Servlet)这类的液体货物, 那就需要使用一个桶(Web容器)装好之后, 再运输.

Web容器实现了**J2EE**中关于网络组件(Web component)[相关规范](https://en.wikipedia.org/wiki/Java_servlet), 该规范为Web组件定义了运行时环境相关信息和配置, 包括安全性, 并发性, 生命周期管理, 事务, 部署和其他服务. Web容器处理对servlet,JavaServer Pages(JSP)文件以及包含服务器端代码的其他类型文件的请求. Web容器创建servlet实例,加载和卸载servlet,创建和管理请求和响应对象,以及执行其他servlet管理任务.

## Web容器的种类
### 开源Web容器
开源的Web容器有Apache Tomcat, Apache Geronimo, Jetty, GlassFish, Jaminid, Enhydra, Payara, Winstone, Tiny Java Web Server, WildFly等.这里简要介绍其中有代表性的几种: Tomcat, Jetty.

### 商业Web容器
JBoss Enterprise Application Platform(Red Hat), WebSphere Application Server(IBM), WebLogic Application Server(Oracle), iPlanet Web Server(Oracle), JRun(Adobe System), Orion Application Server(IronFlare), Resin Pro(Caucho Technology), ServletExec(New Atlanta Communications), SAP NetWeaver, tcServer(SpringSource Inc).

这里主要介绍其中比较有代表性的WebSphere, JBoss.

## 实际操作
这里实际操作, 以开源tomcat进行本地安装测试.

### 下载安装
下载地址: https://tomcat.apache.org/


## 附录
[wikipedia](https://en.wikipedia.org/wiki/Web_container)

