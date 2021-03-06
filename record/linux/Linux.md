# Linux 笔记

## 腾讯云关闭监控

如果使用的是腾讯云服务器, 默认会安装安全监控脚本, 并在`crontab`中设置自动运行, 关闭方法:

- 卸载程序(执行下列脚本):

  ```java
  /usr/local/qcloud/stargate/admin/uninstall.sh
  /usr/local/qcloud/YunJing/uninst.sh
  /usr/local/qcloud/monitor/barad/admin/uninstall.sh
  ```

- 删除对应文件夹: `rm -rf /usr/local/qcloud`.

- 清除定期执行计划: `crontab -r`(注,如果之前自己设置了定期计划, 就需要使用`crontab -e`手动选择删除).

- 查看是否运行: `ps -A | grep agent`.

## Nginx 相关操作

## 安装

`yum install nginx`.

### 校验配置文件

- 校验配置文件是否正确: `nginx -t`.
- 重新加载配置文件: `nginx -s reload`.

### 使用`certbot`进行加密

#### 安装证书

使用免费的 certbot 进行安装：[官方网址](https://certbot.eff.org/)

`Nginx/Centos7`示例:

- 安装依赖: `yum -y install yum-utils`
- 添加 yum 配置: `yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional`
- 安装软件: `sudo yum install certbot python2-certbot-nginx`.
- 运行软件: `sudo certbot --nginx`(注该命令会自动更新 nginx 配置文件, 如果需要自定义的话使用`sudo certbot --nginx certonly`).

#### 更新证书

一次安装持续 90 天,到期时需要手动进行更新: `certbot renew --dry-run`. 这里需要注意的是: **Certbot renew 的时候要检查一下 nginx 的配置文件中的 server_name, 注意要保证前后一致（即创建 SSL 时和更新时要一致），因为创建 SSL 时会新建一个 server block，更新时会检查，如果不一致就会更新失败.**

如果上述命令成功执行, 则可以进行更新`certbot renew`.

添加自动更新, 一天执行两次凌晨进行更新检测`crontab -e`:

```java
0 0,12 * * * python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew
```

#### 转换证书

默认生成的证书为`pem`类型的证书, 如果需要转换为`jks`证书用于相关 Java 后端程序:

```java
//拷贝证书到本地
 scp -r root@193.112.205.181:/etc/letsencrypt/live/cjyong.com ./
//使用openssl转换为PKCS12类型证书
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -out fullchain_and_key.p12 -name cjyong -passout pass:cai123nb
//使用keystool生成jks类型证书
keytool -importkeystore -srckeystore fullchain_and_key.p12 -srcstoretype PKCS12 -srcstorepass cai123nb -deststoretype JKS -destkeystore cjyong.jks -deststorepass cai123nb -alias cjyong
```

### 配置 X-Frame-Options 头

配置 X-Frame-Options 头可以防止网站被嵌入到别的网站中的 Frame 中进行劫持攻击. 配置文件: `/etc/nginx/conf.d/default.conf`. 在 Server 下配置:

```java
//设置只有在相同源地址时才可以进行嵌入
add_header X-Frame-Options SAMEORIGIN;
```

## Redis 相关操作

- 进入命令行: `redis-cli`.
- 授权登录: `auth PASSWORD`
- 查看所有的KEY: `KEYS *`
- 详细密码列表:[官方文档](https://redis.io/commands)

### Redis 序列化失败问题

当出现序列化失败时, 检查序列化的对象是否实现了`Serializable`, 并显式设置了`serialVersionUID`(如果不显式设置, 修改之后将会自动生成新的 UID, 将不会兼容之前的缓存).

清除缓存: `redis-cli, flushdb/flushall`

### Redis keys无效

当我们使用`keys *`查看所有的key时没有效果.往往是当前数据库不对. 查看和切换数据库的方法:

- `CONFIG GET databases`: 查看当前几个数据库.
- `info keyspace`: 查看那几个数据库存在key(存在使用情况)
- `select num`: 选择你要去的数据库, 如`select 2`.
- `keys *`: 查看所有key.

## Docker 相关操作

### Docker 基本指令

- `docker login`: docker 用户登录, 拉取前需要先登录, 否则会授权失败.

- `docker search redis`: 搜索名字为`redis`的镜像.

- `docker pull redis`: 拉取名字为`redis`的镜像, 未添加版本号, 默认使用最新版本(`docker pull redis:3.2`).

- `docker images`: 查看之前构建或者拉取的所有镜像.

- `docker run -p 6379:6379 -d redis:latest myredis`: 运行 redis 镜像, 生成容器并启动.

- `docker ps`: 查看所有运行容器, 添加`-a`则可以查看所有容器(包括关闭的容器).

- `docker start/stop e6`: 启动/关闭前缀为`e6`的容器.

- `docker exec -it e6 redis-cli`: 以交互的方式连接`e6`容器, 并执行`redis-cli`命令.

## JVM 程序监控

### 常用的工具

自带: `jconsole`, `jvisualvm`, `jstack`.

### 示例

示例 1: `统计Java程序线程数量(按照状态划分)`:

```java
jstack 26173 > dump173

grep java.lang.Thread.State dump173 | awk '{print $2$3$4$5}' | sort | uniq -c
```

## Spring Boot 项目配置为 Service

```c
//笔者的Linux系统为Centos7,系统之间存在差异,请酌情修改
1\. 创建自己的service文件

cd /usr/lib/systemd/system/
vim myServe.service

2\. 写入配置信息

[Unit]
Description=My personal Service
After=syslog.target

[Service]
ExecStart=/usr/local/java/jdk1.8/bin/java -jar /root/.m2/repository/com/cjyong/cp/personalweb/0.0.1-SNAPSHOT/personalweb-0.0.1-SNAPSHOT.jar
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target

3\. 启动服务

service myServe start

4\. 注册开机启动

systemctl enable myservice.service
```

## 磁盘内文件大小的管理

主要的命令`du`, `du`(Disk Usage): 查看文件或者目录的占用空间.

常用命令:

`du -h`: 查看当前文件夹内文件的使用情况 `du -h filename`: 查看 filename 的文件夹下使用情况. `du -ah filename`: 查看 filename 的文件夹及其子文件夹和文件使用情况 `df -h`: 查看磁盘的使用情况 `echo "" > filename`: 清空 filename 文件内容

## travis CI 使用

准备条件: 在 travis ci 中进行登录并绑定你需要 CI 的项目, 根据文档添加.travis.yml 文件.

### 安装 git 和 maven(服务器端)

- `yum install git maven`: 安装 git 和 maven
- `git clone xx`: 拷贝你的项目

### 安装 travis(服务器端)

- `yum install gem` : 安装 gem
- `gem sources -l` : 列出源路径
- `gem sources --remove xxxx`: 移除国外路径
- `gem sources --add https://gems.ruby-china.com/` : 添加国内路径
- `gem install travis` : 安装 travis, 如果出现问题, 尝试使用`yum install ruby ruby-devel ruby-docs ruby-ri ruby-rdoc rubygems`补全依赖的东西, 更新 gcc: `yum install gcc`, 如果还不行, 查看错误日志: `find / -name xxx.log`查找日志位置(如果没有给出的话), 'cat/more/vim xx/xxx.log'查看日志,寻找原因进行解决.

### 生成密钥(服务器端)

- `travis login --auto`: 登录你的 git 账号,同步信息
- `ssh-keygen -t rsa`: 生成密钥(不用输入一直 enter,默认存储在/root/.ssh/id_rsa(私钥), 公钥 id_rsa.pub)
- `cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`: 将公钥放入服务器的可信任列表中
- `travis encrypt-file ~/.ssh/id_rsa --add`: 这时候会在.travis.yml 中自动生成:

```java
before_install:
  - openssl aes-256-cbc -K $encrypted_a65ab4f4a956_key -iv $encrypted_a65ab4f4a956_iv
    -in id_rsa.enc -out ~/.ssh/id_rsa -d
  - chmod 600 ~/.ssh/id_rsa
```

**注意,默认生成为`~\/.ssh/id_rsa -d`需要把那个斜杠去掉, 最后为`~/.ssh/id_rsa -d`,并在底部添加对应的权限`- chmod 600 ~/.ssh/id_rsa`, 否则会报错.**

- '添加权限·： 在上面配置的下面添加： `- chmod 600 ~/.ssh/id_rsa`.
- `添加信任站点`: 项目.travis.yml 中添加(ip 地址为你的项目地址)

```java
addons:
   ssh_known_hosts: 193.112.205.181
```

### .travis.yml 示例

```java
language: java
cache:
   directories:
   - .autoconf
   - $HOME/.m2
sudo: true
jdk:
- openjdk8
before_install:
- openssl aes-256-cbc -K $encrypted_a65ab4f4a956_key -iv $encrypted_a65ab4f4a956_iv
   -in id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- chmod +x ./mvnw
script:
- ./mvnw test -B
addons:
   ssh_known_hosts: 193.112.205.181
after_success:
- ssh root@193.112.205.181 "sh /cjyong/project/script/pc.sh"
notifications:
   email:
   recipients:
   - 2686600303@qq.com
   on_success: always
   on_failure: always
```

pc.sh:

```java
cd /cjyong/project/personalcoupleweb/ && git pull origin master && service pcService stop && rm -rf /root/.m2/repository/com/cjyong/cp/personalweb/0.0.1-SNAPSHOT/personalweb-0.0.1-SNAPSHOT.jar && mvn clean install && service pcService start
```
