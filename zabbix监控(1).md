# zabbix监控

标签（空格分隔）： 监控

---

[TOC]


[参考网站][1]

##Zabbix体系概述

1.	硬件监控
2.	系统监控 CPU、内存、IO
3.	应用监控 nginx、apache、mysql、redis、memcache
4.	安全监控
5.	WEB监控  响应时间、下载带宽、加载时间
6.	业务监控
7.	自动化监控  必须安装客户端指向服务端，服务必须运行，识别主机名或者IP 自动加入监控，附上基础监控模板
items 图形 触发器->模板
8.	分布式监控 总服务器和代理机
硬件：
监控对象的指标： CPU
用户态60-70% 内核态0-35% 空闲态0-5%
1核4线程   低于12
2核8线程   低于48
cpu负载 、cpu用户态内核态、cpu使用率（超出安全范围报警）

Linux下常用监控CPU性能的工具有
1. iostat      只能查看所有CPU的平均信息
2. vmstat      能查看所有CPU的平均信息，能查看CPU队列信息
3. mpstat      能查看单个和所有的CPU信息。
4. sar         与mpstat类似
5. top
6. nmon
7. iftop -n    
8. glance 

监控对象的指标：内存
使用率大于70%报警 寻址

  
软件：
   软件有自带的监控

------------------------------------------------------

1.安装zabbix 和 数据库

```
#rpm -ivh https://mirrors.aliyun.com/zabbix/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-1.el7.centos.noarch.rpm
#yum install zabbix-server-mysql zabbix-web-mysql zabbix-server zabbix-agent mariadb-server
```
2.启动ｚａｂｂｉｘ前需要先建立数据库做授权

```
[root@localhost yum.repos.d]# systemctl start mariadb
[root@localhost yum.repos.d]# mysql -uroot -p
Enter password: 
MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;
Query OK, 1 row affected (0.00 sec)　＃设置字符集

MariaDB [(none)]> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';　　　　　　　　　　　　　　＃建立ｚａｂｂｉｘ登陆用户
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
| zabbix             |
+--------------------+
5 rows in set (0.00 sec)

MariaDB [(none)]> quit;
```

3.导入zabbix的items 图形 触发器->模板

```
[root@localhost yum.repos.d]# cd /usr/share/doc/zabbix-server-mysql-3.4.5/
[root@localhost zabbix-server-mysql-3.4.5]# zcat create.sql.gz | mysql -uroot zabbix -p
Enter password: 　　　　＃导入初始模式和数据到数据库
[root@localhost zabbix-server-mysql-3.4.5]# mysql -uroot -p
Enter password: 

MariaDB [(none)]> use zabbix;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [zabbix]> show tables;
+----------------------------+
| Tables_in_zabbix           |
+----------------------------+
| acknowledges               |
| actions                    |
| alerts                     |
+----------------------------+
```
mysql> show databases;
mysql> use zabbix;
mysql> update dbversion set mandatory=3040000;
mysql> flush privileges;

4.编辑zabbix配置文件

```
# vi /etc/zabbix/zabbix_server.conf 
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
```
5.启动zabbix-server

```
[root@localhost zabbix-server-mysql-3.4.5]#systemctl start zabbix-server
```

6.编辑Zabbix前端的PHP配置
```
//Zabbix前端的Apache配置文件位于 /etc/httpd/conf.d/zabbix.conf 一些PHP设置已经完成了配置。

php_value max_execution_time 300
php_value memory_limit 128M
php_value post_max_size 16M
php_value upload_max_filesize 2M
php_value max_input_time 300
php_value always_populate_raw_post_data -1
php_value date.timezone Asia/ShangHai
依据所在时区，你可以取消 “date.timezone” 设置的注释，并正确配置它。在配置文件更改后，需要重启Apache Web服务器。

//重启httpd服务
[root@localhost ／]#systemctl start httpd
```
Zabbix前端可以在浏览器中通过 `http://zabbix-frontend-hostname/zabbix` 进行访问。默认的用户名／密码为 `Admin/zabbix`


选择ＭＹＳＱＬ
按照提示下一步
直到
Congratulations! You have successfully installed Zabbix frontend.
Configuration file "/etc/zabbix/web/zabbix.conf.php" created.
用户名：Admin
密码： zabbix

zabbix启动自起需要开启的服务
pkill nginx
```
[root@localhost ~]# systemctl start zabbix-server
[root@localhost ~]# systemctl enable zabbix-server
[root@localhost ~]# systemctl start httpd
[root@localhost ~]# systemctl enable httpd
[root@localhost ~]# systemctl start mariadb
[root@localhost ~]# systemctl enable mariadb
```
zabbix解决乱码中文乱码

```
[root@localhost fonts]# sed -i 's#graphfont#simkai#g' /usr/share/zabbix/include/defines.inc.php
[root@localhost fonts]# pwd
/usr/share/zabbix/fonts
[root@localhost fonts]# ls
graphfont.ttf  simkai.ttf
[root@localhost fonts]# 
```

##Zabbix客户端配置  

客户端的配置改几行就行了
StartAgents=0    设置0为使用主动模式
ServerActive=zabbixserver地址
Hostname=主机名和zabbix中添加的服务器名字一致


1.配置nginx、php客户端的服务打开状态信息，重启服务；
2.通过编写脚本获取到相关状态值；
3.再通过配置文件，监控获取到的key值
4.分别将脚本存放到/etc/zabbix/scripts目录   ，conf存放在/etc/zabbix/zabbix_agentd.d目录
5.将修改号的脚本和监控项目录，拷贝到每台机器指定的位置
6.重启所有机器的zabbix-agent
7.再server端使用zabbix_get获取每台服务器监控项的值
8.再zabbix-web界面导入xml模板
9.再到每台服务器中添加模板，

1.服务端和客户端都安装zabbix-agent客户端服务  
  （监控模式除了agent 还有 SSH ，本文只讨论agent的监控模式）
```
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-5.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
rpm -ivh https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/rhel/7/x86_64/zabbix-agent-3.4.5-1.el7.x86_64.rpm
```
2.编辑配置文件修改服务端IP属性 并启动服务

```
vim /etc/zabbix/zabbix_agentd.conf
97 server=192.168.56.134 
多台设备可以直接拷贝过去 #scp /etc/zabbix/zabbix_agentd.conf root@192.168.56.11:/etc/zabbix/
systemctl start zabbix-agent ; netstat -plant
10050 
```

登录之后 最上放点击 配置 --> 主机 --> 创建主机
1.主机名称与agent代理程序的接口 需要填写IP或者域名
其他可以更具需求配置
http://m.qpic.cn/psb?/V11L9NoS163zXU/PKIX2OJD.XB4lNY6xuSmXtkbUJD.4GG\*xHVfM9mPfks!/b/dPIAAAAAAAAA&bo=gAKZAgAAAAARByk!&rf=viewer_4
2.模板页面 搜索linux 选择Template OS Linux模板 点击小添加
配置完成 如果需要监控多台 就如上克隆多台主机


##zabbix自定义报警
当前登录终端用户不能超过1个，如果超过就报警：
1.首先在客户机上查询有几个用户登录：

```
/usr/bin/w|awk 'NR==1{print $4}' 
```
2.agent的key如何传输到server ？（key:监控对象的指标）
在客户机上自定义key： 

```
vi  /etc/zabbix/zabbix_agentd.conf
   UserParameter=log_user,/usr/bin/w|awk 'NR==1{print $4}' 
systemctl restart zabbix-agent  
```
3.主机上如何验证key，实际工作时没有验证。

```
yum -y install zabbix-get
zabbix_get -s 192.168.56.11 -p10050 -k log_user   
```
4.zabbix的web页面配置监控项
在zabbix中配置客户机的items
选择客户机 --> 监控项 
http://m.qpic.cn/psb?/V11L9NoS163zXU/q6\*VH5c61F163oSB.areOZoA2N\*IHunAObil9bghs00!/b/dPIAAAAAAAAA&bo=gAINAwAAAAARB7w!&rf=viewer_4
在zabbix中配置客户机的监控图形
http://m.qpic.cn/psb?/V11L9NoS163zXU/D9m7PY1VoTNTvV7vgPDMBh2sb6J2Sp2H.IDT0SeoY0Q!/b/dPMAAAAAAAAA&bo=8gMnAgAAAAARB.Q!&rf=viewer_4

5.配置触发器
触发器报警也仅仅只在zabbix-web页面报警
名称： login_user
表达式： 最新的T值等于N  （{192.168.56.11:login_user.last()}>1）
动作执行的流程->
          判断符合条件的服务器
          执行动作->发消息或者执行命令
          用什么方式发送（mail） -> 发给那个组（DBA|OPS）
          具体发给那个用户
http://m.qpic.cn/psb?/V11L9NoS163zXU/6KqWI2bhKPYM6dotqTxJApN\*KIWLWTstGewkQ0NxL7w!/b/dPIAAAAAAAAA&bo=qQKAAgAAAAARBxk!&rf=viewer_4


6.配置好主机监控后会在 配置 --> 主机 面板看到可用性 ZBX 变绿
每一台都需要此类配置

7.如果没有变绿可以查看日志文件tail -f /var/log/zabbix

##zabbix配置邮件通知

1.创建报警媒介media
http://m.qpic.cn/psb?/V11L9NoS163zXU/GZ.fMYi18FQES43HHb\*g6pxlfhst.aDgh2.Dt5C7lVc!/b/dPMAAAAAAAAA&bo=2gKAAgAAAAARB2o!&rf=viewer_4
注： qq邮箱需要配置第三方密码

2.配置发送警告的用户
http://m.qpic.cn/psb?/V11L9NoS163zXU/duboi10eLxa2TZmPDF9TalhHtYk84\*BWa8ZAMLZ0c5o!/b/dGUBAAAAAAAA&bo=TgWCAQAAAAARB\*g!&rf=viewer_4
配置报警邮箱
http://m.qpic.cn/psb?/V11L9NoS163zXU/elZoj5aSdRNAwzPoyayzYt50C4b7.\*7piRos9FIi158!/b/dFsBAAAAAAAA&bo=DQOAAgAAAAARF6w!&rf=viewer_4
保存并点击状态为开启！

3.添加动作
配置action，需要如下几个步骤：
①.在菜单栏中单击Configuration→Actions。
②.在Event source下拉菜单中选择事件来源。
③.单击Create action。
④.设置Action参数,以及报警消息内容。设置如下：
http://m.qpic.cn/psb?/V11L9NoS163zXU/XeAD20eI2XuKhT69.BCDiyLmKb5AmDISLYNmTs61Epw!/b/dPIAAAAAAAAA&bo=PwO2AQAAAAARB7s!&rf=viewer_4
⑤.单机Conditions按钮，设置Action的依赖条件。
⑥.单击Operation按钮，设置执行动作。设置如下：
http://m.qpic.cn/psb?/V11L9NoS163zXU/oy8de\*G\*Ob.5HxLS.ulyMpU9woeSzfT.4OFhao7STxI!/b/dPIAAAAAAAAA&bo=rQKAAgAAAAARFw0!&rf=viewer_4

4.检测 打开数个SSH登录框报警 查看是否收到邮件
http://m.qpic.cn/psb?/V11L9NoS163zXU/gZ7AqX1cVi6lWFrpvM49rxO1TNVSu0HdDDpMlU8cMBA!/b/dPIAAAAAAAAA&bo=1QOAAgAAAAARB2Q!&rf=viewer_4



##zabbix监控Nginx状态

agent端安装:
1.获取nginx状态信息

```
//安装nginx
 yum -y install nginx 

 //配置nginx开启状态模块
 vim /etc/nginx/nginx.conf

     47         location / {
     48         }
     49         location /nginx_status{
     50                 stub_status on;    #状态信息打开
     51                 access_log off;    #访问日志关闭
     52                 allow 127.0.0.1;
     53                 deny all;
     54     }   
    
//检查nginx语法
 nginx  -t
 
//启动Nginx
 nginx
 
//浏览器验证
 http://192.168.56.11/nginx_status
```
2.手动awk过滤状态信息

```
/usr/bin/curl -s "http://127.0.0.1:"\$NGINX_PORT"/nginx_status/" |awk '/Active/ {print $NF}'
```

3.使用shell脚本处理各个状态信息的值
wget http://fj.xuliangwei.com/zabbix/scripts/nginx_status.sh
4.在/etc/zabbix/创建scripts目录, 用于存放脚本
5.给脚本加上执行权限
```
  345  mkdir scripts
  346  mv ~/nginx_status.sh ./
  347  ls
  348  chmod +x nginx_status.sh 
  349  pwd
  350  mv nginx_status.sh ./scripts/
  351  ls
  352  cd scripts/
  353  ls
  354  systemctl restart zabbix-agent

```
6.自定义监控项  /etc/zabbix/zabbix_agent.d/目录
7.创建nginx_status.conf文件(必须是conf结尾)
UserParameter=nginx_status[*],/bin/bash /etc/zabbix/scripts/nginx_status.sh "$1"
8.在zabbix-server端验证agent端脚本是否正常
zabbix_get -s 192.168.56.12 -p10050 -k nginx_status[active]
9.配置zabbix-server监控项
10.配置模板-在模板添加监控项-图形
11.将模板关联agent主机


##Zabbix监控TCP状态

1.利用脚本`tcp_status.sh`获取`TCP`的状态值；也就是获取`KEY`值；
```
[root@cobbler-node1 ~]# cd /etc/zabbix/scripts/
[root@cobbler-node1 scripts]# ls
nginx_status.sh
[root@cobbler-node1 scripts]# wget http://fj.xuliangwei.com/zabbix/scripts/tcp_status.sh
[root@cobbler-node1 scripts]# ls
nginx_status.sh  tcp_status.sh
[root@cobbler-node1 scripts]# chmod +x tcp_status.sh 
[root@cobbler-node1 scripts]# sh tcp_status.sh 
Usage:CLOSE-WAIT|CLOSED|CLOSING|ESTAB|FIN-WAIT-1|FIN-WAIT-2|LAST-ACK|LISTEN|SYN-RECV SYN-SENT|TIME-WAIT
[root@cobbler-node1 scripts]# sh tcp_status.sh ESTAB
1
```
2.在`/etc/zabbix/zabbix_agentd.d`中编辑配置文件并重启`zabbix`；
```
[root@cobbler-node1 scripts]# cd /etc/zabbix/
scripts/            zabbix_agentd.conf  zabbix_agentd.d/    
[root@cobbler-node1 scripts]# cd /etc/zabbix/zabbix_agentd.d
[root@cobbler-node1 zabbix_agentd.d]# ls
nginx_status.conf  userparameter_mysql.conf
[root@cobbler-node1 zabbix_agentd.d]# wget http://fj.xuliangwei.com/zabbix/zabbix_agentd.d/tcp_status.conf
[root@cobbler-node1 zabbix_agentd.d]# cat tcp_status.conf 
UserParameter=tcp_status[*],/bin/bash /etc/zabbix/scripts/tcp_status.sh "$1"
[root@cobbler-node1 zabbix_agentd.d]# systemctl restart zabbix-agent
[root@cobbler-node1 scripts]# rm /tmp/ss.txt 
rm: remove regular file ‘/tmp/ss.txt’? y
```
3.服务端查看
```
[root@localhost ~]# zabbix_get -s 192.168.56.11 -p10050 -k tcp_status[ESTAB]
2
```
4.zabbix网页 配置 --> 模板 --> 导入 --> 选择提前做好的模板
下载的地址C:\Users\altri\Downloads\tcp_status.xml
5.关联主机  配置 --> 主机 --> 模板 --> 添加上传的模板 --> 更新即可


##Zabbix中文字符集

中文字符集

```
[root@linux-node1 ~]# wget http://download.xuliangwei.com/SIMHEI.ttf
[root@linux-node1 ~]# mv SIMHEI.ttf  /usr/share/zabbix/fonts/
[root@linux-node1 ~]# cd /usr/share/zabbix/fonts/

修改配置文件
vim /usr/share/zabbix/include/defines.inc.php
define('ZBX_GRAPH_FONT_NAME',           'SIMHEI');
define('ZBX_FONT_NAME', 'SIMHEI');

重新加载server配置文件
systemctl restart zabbix-server
```

##PHP-fpm监控
1.先安装php
```
[root@cobbler-node1 scripts]# yum -y install php-fpm php
[root@cobbler-node1 scripts]#systemctl status php-fpm
[root@cobbler-node1 scripts]#netstat -lntup | grep 9000
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      57570/php-fpm: mast
```
2.配置php状态模块
```
[root@cobbler-node1 nginx]# vi /etc/php-fpm.d/www.conf #我php-fpm存放路径
pm.status_path = /phpfpm_status  #打开php状态信息监控
[root@cobbler-node1 nginx]# systemctl restart php-fpm   #重启php- fpm
[root@cobbler-node1 nginx]# vi /etc/nginx/nginx.conf    #关联php状态信息到nginx
   location ~ ^/(phpfpm_status)$ {
        include fastcgi_params;
        fastcgi_pass    127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
[root@cobbler-node1 nginx]# nginx -s reload           #重启nginx
```
3.获取备监控对象的指标
  访问看能否获取状态信息 http://192.168.56.11/phpfpm_status
```
[root@cobbler-node1 nginx]# curl http://192.168.56.11/phpfpm_status
pool:                 www
process manager:      dynamic
start time:           10/Jan/2018:21:52:06 -0500
start since:          190
accepted conn:        1
listen queue:         0
max listen queue:     0
listen queue len:     128
idle processes:       4
active processes:     1
total processes:      5
max active processes: 1
max children reached: 0
slow requests:        0
```
4.写成shell脚本
```
[root@cobbler-node1 nginx]# wget http://fj.xuliangwei.com/zabbix/scripts/phpfpm_status.sh  #下载获取值的脚本，不怕麻烦可以自己写,修改其中端口
[root@cobbler-node1 scripts]# chmod +x phpfpm_status.sh 
[root@cobbler-node1 nginx]# mv phpfpm_status.sh /etc/zabbix/scripts/  #考到zabbix下
```
5.agent端配置自定义key
```
[root@cobbler-node1 zabbix_agentd.d]# wget http://fj.xuliangwei.com/zabbix/zabbix_agentd.d/phpfpm_status.conf  #查看一下
[root@cobbler-node1 zabbix_agentd.d]# systemctl restart zabbix-agent  #重启
```
6.server检测agent获取的key
```
[root@localhost zabbix]# zabbix_get -s 192.168.56.11 -p10050 -k phpfpm_status[start_since]
16

```
7.web页面配置模板
配置 --> 模板 --> 导入 --> 选择提前做好的模板
下载的地址C:\Users\altri\Downloads\phpfpm_status.xml
8.关联主机
 配置 --> 主机 --> 模板 --> 添加上传的模板 --> 更新即可
 
 
##监控tomcat
在Zabbix中，JMX监控数据的获取由专门的代理程序来实现,即Zabbix-Java-Gateway来负责数据的采集，Zabbix-Java-Gateway和JMX的Java程序之间通信获取数据

JMX在Zabbix中的运行流程：
a)	Zabbix-Server找Zabbix-Java-Gateway获取Java数据
b)	Zabbix-Java-Gateway找Java程序(zabbix-agent)获取数据
c)	Java程序返回数据给Zabbix-Java-Gateway
d)	Zabbix-Java-Gateway返回数据给Zabbix-Server
e)	Zabbix-Server进行数据展示
配置JMX监控的步骤：
a)	安装Zabbix-Java-Gateway。
b)	配置zabbix_java_gateway.conf参数。
c)	配置zabbix-server.conf参数。
d)	Tomcat应用开启JMX协议。
e)	ZabbixWeb配置JMX监控的Java应用。


1.客户端下载tomcat
```
wget http://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-9/v9.0.2/bin/apache-tomcat-9.0.2.tar.gz
```
2.客户端解压并启动tomcat
```
[root@localhost ~]# cd /usr/share/apache-tomcat-9.0.2/bin/
[root@localhost bin]# ./startup.sh 
[root@localhost bin]# netstat -plant

```

3.服务端安装java的网关 找到jvm
[root@localhost ~]# yum install  zabbix-java-gateway java-1.8.0-openjdk -y
[root@localhost ~]# systemctl start zabbix-java-gateway
[root@localhost ~]# vi /etc/zabbix/zabbix_server.conf 
StartJavaPollers=5     #启动多少个进程去轮巡
JavaGateway=192.168.56.134      #找java的jvm
[root@localhost ~]# systemctl restart zabbix-server
[root@localhost ~]# netstat -plant | grep java
tcp6       0      0 :::10052                :::*                    LISTEN      12166/java        
 
4.客户端开启jvm远程访问
```
[root@localhost bin]# vi catalina.sh   #开启自己的jvm远程访问
CATALINA_OPTS="$CATALINA_OPTS
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=12345    #从哪个端口进行jvm访问
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=192.168.56.135"
[root@localhost bin]# ./shutdown.sh
[root@localhost bin]#  ./startup.sh
[root@localhost bin]# netstat -plant
tcp6       0      0 :::12345                :::*                    LISTEN      5957/java
```

###AB压力测试
ab 的用法是：ab [options] [http://]hostname[:port]/path
例如：ab -n 5000 -c 200 http://localhost/index.php
上例表示总共访问http://localhost/index.php这个脚本5000次，200并发同时执行。
ab常用参数的介绍：
-n ：总共的请求执行数，缺省是1；
-c： 并发数，缺省是1；
-t：测试所进行的总时间，秒为单位，缺省50000s
-p：POST时的数据文件
-w: 以HTML表的格式输出结果

###目标
最基本的cpu、内存、磁盘、网络  Template OS Linux 
| proxy机器监控  | 监控tcp、nginx |
| -
| web+php两台    | 监控tcp、nginx、php-fpm |
| nfs服务器	     | 监控tcp、端口、tcp的2049端口、udp111 |
| rsync服务器    |  监控tcp、873端口|
| mysql服务器	 |  监控tcp、使用percona监控mysql|


zabbix-server和zabbix-agent彻底分离







  [1]: https://www.zabbix.com/documentation/3.4/zh/manual/installation/install_from_packages