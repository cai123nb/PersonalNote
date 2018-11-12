## 磁盘内文件大小的管理
主要的命令`du`, `du`(Disk Usage): 查看文件或者目录的占用空间.

常用命令: 
`du -h`: 查看当前文件夹内文件的使用情况
`du -h filename`: 查看filename的文件夹下使用情况.
`du -ah filename`: 查看filename的文件夹及其子文件夹和文件使用情况
`df -h`: 查看磁盘的使用情况
`echo "" > filename`: 清空filename文件内容

## Nginx配置X-Frame-Options头
配置X-Frame-Options头可以防止网站被嵌入到别的网站中的Frame中进行劫持攻击. 配置文件: /etc/nginx/conf.d/default.conf
在Server下配置:

``` 
//设置只有在相同源地址时才可以进行嵌入
add_header X-Frame-Options SAMEORIGIN;
```


## travis CI使用
准备条件: 在travis ci中进行登录并绑定你需要CI的项目, 根据文档添加.travis.yml文件.

### 安装git和maven(服务器端)
+ `yum install git maven`: 安装git和maven
+ `git clone xx`: 拷贝你的项目

### 安装travis(服务器端)
+ `yum install gem` : 安装gem
+ `gem sources -l` : 列出源路径
+ `gem sources --remoeve xxxx`: 移除国外路径
+ `gem sources --add https://gems.ruby-china.org/` : 添加国内路径
+ `gem install travis` : 安装travis, 如果出现问题, 尝试使用`yum install ruby ruby-devel ruby-docs ruby-ri ruby-rdoc rubygems`补全依赖的东西, 更新gcc: `yum install gcc`, 如果还不行, 查看错误日志: `find / -name xxx.log`查找日志位置(如果没有给出的话), 'cat/more/vim xx/xxx.log'查看日志,寻找原因进行解决.

### 生成密钥(服务器端)
+ `travis login --auto`: 登录你的git账号,同步信息
+ `ssh-keygen -t rsa`: 生成密钥(不用输入一直enter,默认存储在/root/.ssh/id_rsa(私钥), 公钥id_rsa.pub)
+ `cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`: 将公钥放入服务器的可信任列表中
+ `travis encrypt-file ~/.ssh/id_rsa --add`: 这时候会在.travis.yml中自动生成: 
	```
	before_install:
	  - openssl aes-256-cbc -K $encrypted_a65ab4f4a956_key -iv $encrypted_a65ab4f4a956_iv
      -in id_rsa.enc -out ~/.ssh/id_rsa -d
	```
+ ‘添加权限·： 在上面配置的下面添加： `- chmod 600 ~/.ssh/id_rsa`.
+ `添加信任站点`: 项目.travis.yml中添加(ip地址为你的项目地址)
	```
	addons:
	  ssh_known_hosts: 193.112.205.181
	```
### .travis.yml示例

	```
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
	
	```
	cd /cjyong/project/personalcoupleweb/ && git pull origin master && service pcService stop && rm -rf /root/.m2/repository/com/cjyong/cp/personalweb/0.0.1-SNAPSHOT/personalweb-0.0.1-SNAPSHOT.jar && mvn clean install && service pcService start
	```

