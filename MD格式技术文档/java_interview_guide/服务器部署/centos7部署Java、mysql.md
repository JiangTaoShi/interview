### 0、学习目标

- 阿里云服务器

- 命令回顾

- 安装jdk8

- 安装mysql

  

### 1、阿里云服务器

#### 1、 选择服务器

打开网址：`https://www.aliyun.com/minisite/goods?userCode=u3b3p3wa`

- 注册
- 登陆
- 选择购买
- 配置ssh连接密码



#### 2、配置安全组

开端口

mysql   3306

http		80



![](http://api.zc16.top/server/../Public/Uploads/2020-07-06/5f02d1d520ac9.png)





### 2、安装Java环境

环境变量：通过yum命令安装的jdk不需要手动配置环境变量

#### 1、安装

```
yum -y install java-1.8.0-openjdk*
```

#### 2、验证：

```
java -version
```

#### 3、上传项目并运行(前台运行)

```
java -jar xxx.jar
ctrl + c停止
```

#### 4、后台运行项目

```
nohup java -jar xxx.jar > /dev/null 2>&1 &
```

这个命令会返回一个pid，就是项目的进程id

```
kill -9 pid
```



### 3、安装msyql

需要关注：配置文件，/etc/my.cnf

<span style='color:red;'>写在前面：如果要外网连接，必须配置第9步和开放3306端口 </span>

#### 1、安装yum源文件

```
rpm -Uvh https://repo.mysql.com//mysql80-community-release-el7-3.noarch.rpm
```

![](http://api.zc16.top/server/../Public/Uploads/2020-07-05/5f01ba39003da.png)

#### 2、查看当前默认下载的信息

```
yum repolist enabled | grep "mysql.*-community.*"
```
![](http://api.zc16.top/server/../Public/Uploads/2020-07-05/5f01ba58c86ba.png)

#### 3、修改要下载的版本，默认是8.0的

```
vim /etc/yum.repos.d/mysql-community.repo
```
![](http://api.zc16.top/server/../Public/Uploads/2020-07-05/5f01bab7338fb.png)

#### 4、再次查看版本信息，就是5.6的啦

```
yum repolist enabled | grep "mysql.*-community.*"
```
![](http://api.zc16.top/server/../Public/Uploads/2020-07-05/5f01baf30919a.png)

#### 5、安装

```
yum -y install mysql-community-server
```
安装过程中yum会自己检测依赖包下载安装，等待安装完成即可

#### 6、启动、查看状态、添加开机自启

```
systemctl start mysqld         	启动
systemctl stop mysqld        	停止
systemctl enable mysqld   	添加开机自启
systemctl status mysqld      	查看mysql状态
```

#### 7、创建用户并登陆(mysql启动之后)

```
mysqladmin -uroot password root 		创建root用户
mysql -uroot -proot         					登陆mysql，控制台临时登陆
```

#### 8、允许root远程访问
在msyql登陆之后的mysql命令行中，依次执行如下命令

```sql
grant all privileges on *.* to 'root'@'%' identified by 'root' with grant option;
flush privileges;
```

#### 9、修改配置文件

`vi /etc/my.cnf`

根据需要，配置成如下样子，[mysqld][client][mysql]这三个下的东西必须匹配
```
[mysqld]
character_set_server=utf8mb4
collation-server = utf8mb4_unicode_ci
#设置二进制日志
log_bin=binary-log
[client]
default-character-set=utf8mb4
[mysql]
default-character-set = utf8mb4

```



###  4、yum升级msyql



#### 1、备份原有数据库

原有数据库位置在：`/var/lib/mysql`文件夹下

所以先备份这里的文件内容：`mv /var/lib/mysql /其他目录`

如果原有数据库存的内容不重要，直接删除即可：`rm -rf /var/lib/mysql/*`



#### 2、修改yum源当前默认版本

按照安装过程中的第3步，把想要安装的mysql的版本的enabled改为1，其他的改为0即可



#### 3、更新

输入命令:

```
yum -y update mysql-community-server
```

然后，喝口水等一下即可

![](http://api.zc16.top/server/../Public/Uploads/2020-07-07/5f04067d0ab6f.png)





#### 4、修改配置文件

因为在上边安装过程中，在my.cnf中配置了：`log_bin=binary-log`这一项，然后需要在加一行：`server_id=1`

还有就是需要改一下sql_mode的值，5.7版本会引发sql_mode的坑！

以下是最新配置文件的样子：

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

character_set_server=utf8mb4
collation-server = utf8mb4_unicode_ci
log_bin=binary-log
server_id=1
symbolic-links=0

sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid


[client]
default-character-set=utf8mb4
[mysql]
default-character-set = utf8mb4

```





#### 5、启动、查看状态、添加开机自启

```
systemctl start mysqld         	启动
systemctl stop mysqld        	停止
systemctl enable mysqld   	添加开机自启
systemctl status mysqld      	查看mysql状态
```

#### <span style="color:red;">6、修改root用户密码</span>

5.7版本的mysql启动完了之后，很蛋疼的生成了一个非常复杂的密码，在运行日志里才能找到，按照如下步骤操作，找到那个密码：



方式一：（阿里云centos6.4推荐这种方式）

运行：`journalctl -xe | grep "A temporary"` 

![](http://api.zc16.top/server/../Public/Uploads/2020-07-07/5f04201588e2c.png)

出来的红框里的就是密码，复制好



--------------------



方式二：

`grep "A temporary" /var/log/mysqld.log`

同样会出现上面的图片内容，但是在新买的服务器上没有找到，所以使用方式一吧



复制好密码之后，连接mysql

```
mysql -uroot -p
```

然后回车，提示你输入密码，就把刚才复制的密码粘贴上，回车，注意：这里粘贴之后之不会有任何效果，直接回车就行

然后就进入msyql了

![](http://api.zc16.top/server/../Public/Uploads/2020-07-07/5f04276672200.png)



#### 7、修改密码(<span style="color:red">必须修改密码才能操作其他的</span>)

mysql5.7接着蛋疼的点，就是密码强度问题

![](http://api.zc16.top/server/../Public/Uploads/2020-07-07/5f0427c63bab5.png)

默认是上图中的1，就是中等强度，必须是带数字、字母、特殊字符等的8位以上的

所以，要想改成简单的123456这样的密码，必须把密码强度和长度都改了，在mysql的依次命令行输入：

```linux
set global validate_password_policy=0;   
set global validate_password_length=1;
```

然后就可以修改成简单的密码了，在mysql的依次命令行输入：

```
alter user user() identified by 'root';
```



#### 8、允许root远程访问
在msyql登陆之后的mysql命令行中，依次执行如下命令

```sql
grant all privileges on *.* to 'root'@'%' identified by 'root' with grant option;
flush privileges;
```



到此为止，你就可以开心的连接你的mysql5.7了