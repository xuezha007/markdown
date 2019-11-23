[TOC]



#### 常见脚本 linux

#####  在线安装MySQL数据库

* 替换yum源

  ```shell
  curl -o /etc/yum.repos.d/CentOS-Base.repo mirrors.163.com/.help/CentOS7-Base-163.repo
  ```

  ```shell
  yum clean all 
  yum makecache 
  ```

* 下载rpm文件

  ```shell
  yum localinstall https://repo.mysql.com//mysql80-community-release-el7-1.noarch.rpm
  ```

* 安装MySQL数据库

  ```shell
  yum install mysql-community-server -y
  ```

#####  本地安装MySQL数据库

* 把本课程git工程里共享的MySQL本地安装文件上传到Linux主机的/root/mysql目录

* 执行解压缩

  ```shell
  tar xvf mysql-8.0.11-1.el7.x86_64.rpm-bundle.tar
  ```

* 安装依赖的程序包

  ```shell
  yum install perl -y
  yum install net-tools -y
  ```

* 卸载mariadb程序包

  ```shell
  rpm -qa|grep mariadb
  rpm -e mariadb-libs-5.5.60-1.el7_5.x86_64 --nodeps
  ```

* 安装MySQL程序包

  ```shell
  rpm -ivh mysql-community-common-8.0.11-1.el7.x86_64.rpm 
  rpm -ivh mysql-community-libs-8.0.11-1.el7.x86_64.rpm 
  rpm -ivh mysql-community-client-8.0.11-1.el7.x86_64.rpm 
  rpm -ivh mysql-community-server-8.0.11-1.el7.x86_64.rpm 
  ```

* 修改MySQL目录权限

  ```shell
  chmod -R 777 /var/lib/mysql/
  ```

* 初始化MySQL

  ```shell
  mysqld --initialize
  chmod -R 777 /var/lib/mysql/*
  ```

  

* 启动MySQL

  ```shell
  service mysqld start
  ```

* 查看初始密码

  ```shell
  grep 'temporary password' /var/log/mysqld.log
  ```

* 登陆数据库之后，修改默认密码

  ```mysql
  alter user user() identified by "mysql"; 
  ```

* 允许远程使用root帐户

  ```mysql
  UPDATE user SET host = '%' WHERE user ='root';
  FLUSH PRIVILEGES;
  ```

* 允许远程访问MySQL数据库（/etc/my.cnf）

  ```ini
  character_set_server = utf8
  bind-address = 0.0.0.0
  
  ```

* 重启服务器

    ```
    service mysqld restart
    ```

    

* 开启防火墙3360端口

  ```shell
  firewall-cmd --zone=public --add-port=3306/tcp --permanent
  firewall-cmd --reload
  ```





#### 在线修改表结构

在业务系统运行的过程中随意删改字段，会造成重大事故。常规的做法是业务停机，维护表结构，但是不影响正常业务的表结构是允许在线修改的。

【ALTER TABLE 修改表结构的弊病】

* 由于修改表结构是表级锁，因此在修改表结构时，影响表写入操作
* 如果修改表结构失败，必须还原表结构，所以耗时更长
* 大数据表记录多，修改表结构锁表时间很久

【使用Percona-Toolkit工具】

* 安装第三方依赖包

  ```shell
  yum install  -y perl-DBI
  yum install  -y perl-DBD-mysql
  yum install  -y perl-IO-Socket-SSL
  yum install  -y perl-Digest-MD5
  yum install  -y perl-TermReadKey
  ```

* 安装Percona-Toolkit

  ```shell
  #进入到Percona-Tookit离线文件所在的目录
  rpm -ivh *.rpm
  ```

* 把客户收货地址表中的name字段改成VARCHAR(20)

  ```shell
  pt-online-schema-change --host=192.168.99.202 --port=3306 --user=root --password=abc123456 --alter "MODIFY name VARCHAR(20) NOT NULL COMMENT '收货人'" D=neti, t=t_customer_address --print --execute
  ```

### 架设本地图床服务器

* 设置Nginx安装源

  ```shell
  rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
  ```

  ```shell
  yum install -y nginx
  ```

* 防火墙开放80端口

  ```shell
  firewall-cmd --zone=public --add-port=80/tcp --permanent
  firewall-cmd --reload
  ```

* 在/usr/share/nginx/html目录下创建文件夹，上传图片即可

###  读多写少和写多读少

#### 读多写少的解决方案

可以把MySQL组建集群，并且设置上读写分离

#### 写多读少的解决方案

如果是低价值的数据，可以采用NoSQL数据库来存储这些数据

如果是高价值的数据，可以用TokuDB来保存



###  海量记录快速分页

利用主键索引来加速分页查询

```mysql
SELECT * FROM t_test WHERE id>=5000000 LIMIT 100;
SELECT * FROM t_test WHERE id>=5000000 AND id<=5000000+100;
```

如果用物理删除，主键不连续，就不能用主键索引来加速分页，所以只能使用折中的方案

```mysql
SELECT t.id, t.name FROM t_test t JOIN ( SELECT id FROM t_test LIMIT 5000000, 100) tmp ON t.id = tmp.id;
```

#### 解决秒杀业务方法

- 乐观锁
    - 在并发环境下，如果多个客户端访问同一条数据，此时就会产生数据不一致的问题，如何解决，通过加锁的机制，常见的有两种锁，乐观锁和悲观锁，可以在一定程度上解决并发访问。乐观锁，顾名思义，对加锁持有一种乐观的态度，即先进行业务操作，不到最后一步不进行加锁，"乐观"的认为加锁一定会成功的，在最后一步更新数据的时候在进行加锁，乐观锁的实现方式一般为每一条数据加一个版本号，更新数据的时候，比较版本号，系统就知道有没有出现数据的并发更新。如果小于等于当前版本号的更新，都会被放弃。
- redis 
    - 运用redis事务
        - 首先是一个watch命令当事务中的key发现被其他事务修改 结束这个事务
        - 一次性把几条命令打包传过去

### 为什么要放弃存储过程、触发器和自定义函数？

因为在数据库集群的场景里，由于存储过程、触发器和自定义函数都是在本地数据库节点上运行，它们与数据库集群业务产生了冲突，所以为了顾全大局，放弃使用数据库本地编程，甚至连数据库本地生成主键的机制也都放弃了。

###  如何避免偷换交易中的商品信息？

B2B电商平台，通常采用保存历次商品修改信息、降低搜索排名

B2C电商平台，只需要保存历次商品修改信息即可

### 如何抵御XSS攻击？

XSS攻击，是通过在网页上嵌入恶意的JavaScript代码，然后当浏览器渲染DOM组件的时候，这段恶意的脚本就执行了，然后盗取个人账号信息。

第一种，大家都已经知道了，那就是在网页里面植入恶意代码即可。植入过的过程就是你到网站上发帖与回帖。别人看到这个网页，他的账号就被你窃取到了。为什么说XSS攻击跟数据库有关系呢？毕竟发帖回帖的内容，是保存到数据库上的。如果我们对保存到MySQL的数据，先做一下转义，那么将来输出到页面上，那么浏览器就不会当它是标签了，因此也就不会执行恶意代码。

第二种XSS攻击的常用方式，就是发送HTML格式的邮件。因为邮件本身就是一个HTML页面，所以往里面挂恶意代码非常容易。当用户在浏览器上登陆网易邮箱，然后点开这个邮件，那么恶意脚本就开始执行，马上就窃取到你的账户信息。抵御这种XSS攻击，就只能靠电子信箱的运营商，加强邮件内容的过滤，筛选出恶意的脚本。

###  数据库缓存（查询缓存）、程序缓存应该选择哪个？

MySQL的查询缓存结构的是KV的，数据库会把执行过的SELECT结果集缓存到内存里面，KEY是SQL语句，VALUE是结果集。

查询缓存最大的缺点就是，只要用户对数据表修改一条记录，就会让这个数据表的缓存大面积的失效，这就造成的IO压力突然增大，所以最好的办法就是不使用查询缓存，而改用数据缓存，数据缓存是把InnoDB数据表和索引中的一部分记录缓存到内存中。用户更新数据的时候，更新了数据表多少条记录，响应个缓存就更改多少条，并不会出现缓存大面积失效的情况。

数据库的查询缓存因为不可以细颗粒度的设置哪张数据表结果集被缓存，但是程序查询缓存可以详细设置哪一条的SQL的结果集被缓存。所以我们可以避开内容经常变化的数据表，对哪些数据不经常变化的数据表设置查询缓存。

SpringCache技术是Java领域里面比较成熟的缓存方案，使用注解就能管理缓存。结合Redis，可以充分发挥查询缓存的优势。

###  MySQL压力测试

压力测试是针对系统的一种性能测试，但是测试数据与业务逻辑无关，更加简单直接的测试读写性能。

压力测试有4个重要测试指标：TPS、QPS、响应时间和并发量

* QPS是每秒钟处理完的请求数量
* TPS是每秒钟处理完的事务数量
* 响应时间是一次请求的平均处理时间
* 并发量是系统能同时处理的请求数量

安装sysbench工具

```shell
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash 
```

```shell
yum -y install sysbench
```

准备测试数据

```shell
sysbench /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua --mysql-host=192.168.99.202 --mysql-port=3306 --mysql-user=root --mysql-password=abc123456 --oltp-tables-count=10 --oltp-table-size=100000 prepare
```

执行测试

```shell
sysbench /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua --mysql-host=192.168.99.202 --mysql-port=3306 --mysql-user=root --mysql-password=abc123456 --oltp-test-mode=complex --threads=10 --time=300 --report-interval=10 run >> /home/mysysbench.log
```

### 4.2 SQL语句优化

* 不要把SELECT子句写成 SELECT *-增加io负担

  ```mysql
  SELECT * FROM t_emp;
  ```

* 谨慎使用模糊查询 - 不要把百分号写在开头-即使设置了索引-但是由于在开头-会仍然无法使用索引

  ```mysql
  SELECT ename FROM t_emp WHERE ename LIKE '%S%'; #不使用索引
  SELECT ename FROM t_emp WHERE ename LIKE 'S%';
  ```

* 对ORDER BY排序的字段设置索引-加快速度

* 少用IS NULL- 会导致跳过索引-因为索引是二叉树 null值无法排序-会导致无法使用索引 

  ```mysql
  SELECT ename FROM t_emp WHERE comm IS NULL; #不使用索引
  SELECT ename FROM t_emp WHERE comm =-1;
  ```

* 尽量少用 != 运算符-无法利用二叉树-会导致全表扫描

  ```mysql
  SELECT ename FROM t_emp WHERE deptno!=20; #不使用索引
  SELECT ename FROM t_emp WHERE deptno<20 AND deptno>20;
  ```

* 尽量少用 OR 运算符 -or之后的查询会使用全表扫描

  ```mysql
  SELECT ename FROM t_emp WHERE deptno=20 OR deptno=30; #不使用索引
  解决办法 
  SELECT ename FROM t_emp WHERE deptno=20 
  UNION ALL
  SELECT ename FROM t_emp WHERE deptno=30;
  ```

* 尽量少用 IN 和 NOT IN 运算符- 相当于是or

  ```mysql
  SELECT ename FROM t_emp WHERE deptno IN (20,30); #不使用索引
  SELECT ename FROM t_emp WHERE deptno=20 
  UNION ALL
  SELECT ename FROM t_emp WHERE deptno=30;
  ```

* 避免条件语句中的数据类型转换

  ```mysql
  SELECT ename FROM t_emp WHERE deptno='20';
  ```

* 在表达式左侧使用运算符和函数都会让索引失效

  ```mysql
  SELECT ename FROM t_emp WHERE salary*12>=100000; #不使用索引
  正确做法
  SELECT ename FROM t_emp WHERE salary>=100000/12;
  SELECT ename FROM t_emp WHERE year(hiredate)>=2000; #不使用索引
  正确做法
  SELECT ename FROM t_emp 
  WHERE hiredate>='2000-01-01 00:00:00';
  ```

### 4.3 MySQL参数优化

* 最大连接数
  * max_connections是MySQL最大并发连接数，默认值151
  * MySQL允许的最大连接数上限是16384
  * 实际连接数(并发连接数)是最大连接数的85%较为合适
  * show  variables like "max_connection"
  * show status like "max_used_connections"
  * 修改my.conf 设置max-connections 设置一个较大的值 然后 看实际连接数大概占多少 通过不断的调整 使得 满足第三条
* 请求堆栈的大小
  * 堆栈是 当请求进不来 先缓存在堆栈中 然后有空余处理能力时在处理
  * back_log是存放执行请求的堆栈大小，默认值是50
  * 一般堆栈大小设置成最大连接数的1/3
* 修改并发线程数
  * innodb_thread_concurrency代表并发线程数，默认是0  0是没有上限的意思
  * 线程太多也不好 会限制cpu的能力导致处理变慢
  * 并发线程数应该设置为CPU核心数的两倍
* 修改连接超时时间
  * wait-timeout是超时时间，单位是秒
  * 连接默认超时为8小时，连接长期不用又不销毁，浪费资源
* 数据缓存-是自身优化结构的不是数据的缓存
  * innodb_buffer_pool_size是InnoDB的缓存容量，默认是128M
  * InnoDB缓存的大小可以设置为主机内存的70%~80%

### 4.4 慢查询日志 

慢查询日志会把查询耗时超过规定时间的SQL语句记录下来，利用慢查询日志，定位分析性能的瓶颈。

slow_query_log 可以设置慢查询日志的开闭状态

long_query_time 可以规定查询超时的时间，单位是秒

```ini
slow_query_log = ON
long_query_time = 1
```

使用explain进行分析

#### 数据库集群

##### 集群分类

- PXC和Replication
- 前者适合保存少量高价值数据
    - 只有数据同步所有服务器才被认为是成功了
- 后者适合保存大量数据
    - 他是异步方案 有一个节点保存成功就认为是成功了

##### **数据库集群能解决什么问题？**

如果在低并发的情况下，单节点MySQL的读写速度更快。因为在数据库集群中，多个MySQL节点的数据要通过网络同步，所以读写速度不如单节点MySQL

但是在高并发的情况下，大量的读写请求会让单节点MySQL的硬盘无法承受，所以速度会变得很慢。比如说12306最开始上线的时候，大量用户涌入网站购票，结果系统就崩溃了。

高并发恰恰是数据库集群的主场。大量的读写请求会被分散发往多个节点执行。众人拾柴火焰高，多个MySQL执行读写请求，肯定比单节点MySQL快，所以在高负载的情况下，单节点MySQL接近崩溃，反而数据库集群的读写速度更快。

### 如何使用Docker虚拟机

#### Docker镜像-一定要基于linux 只有linux才有独立ip docker程序不支持

Docker的镜像文件，相当于是一个只读层，不能往里面写入数据。我们可以通过编写dockerfile文件，定义需要安装的程序，比如说MySQL、Redis、JDK、Node.js等等，然后执行这个dockerfile文件，创建出镜像文件。

手写dockerfile还是比较麻烦的，所以我们可以在Docker的镜像仓库里面寻找别人已经创建好的镜像，比如说你想部署Java程序，那么就在线下载Java镜像即可，非常简单。

毕竟Docker镜像是只读层，如果我们想要往里面部署层序应该怎么办呢？这个很简单，我们可以给镜像创建一个容器，容器是可读可写的，我们把程序部署在容器里就行了。

#### 5.2.2 Docker容器

我们说的在Docker虚拟机中创建实例，指的就是容器。因为镜像的内容是只读的，想要部署程序，我们需要创建出容器实例，容器的内容是可以读写的，可以用来部署程序。

而且容器之间是完全隔离的，我们不用担心一个容器中部署程序，会影响到另一个容器。就比如说我们在CentOS上直接安装MySQL 8.0，它跟Percona Toolkit有冲突，跟Sysbench也有冲突，所以我们做在线修改表结构，以及做压力测试的时候，都是挑选新的虚拟机实例来安装这些程序，访问MySQL的。如果用上了Docker，我可以在A容器里安装MySQL，在B容器跑压力测试，根本不会有冲突。

再有，必须先有镜像，才能创建出容器，镜像和容器之间是关联的关系。而且一个镜像可以创建出多个容器，像是SaaS云计算，运营商可以把进销存系统打成镜像。有企业购买进销存系统，那么运营商就给客户创建一个容器，客户的进销存数据保存在容器A里面。再有客户购买进销存系统，运营商就创建容器B，以此类推。云计算服务商就是这么卖软件的。

#### 5.2.3 安装Docker虚拟机

```shell
yum install -y docker
```

```shell
service docker start
```

```shell
service docker stop
```

#### 5.2.4 操作Docker虚拟机

![](笔记图片\1.png)

##### 设置镜像加速器

```shell
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

编辑/etc/docker/daemon.json文件，把结尾的逗号去掉

#### 管理镜像

```shell
#搜索镜像
docker search 关键字
#下载镜像
docker pull 镜像名字
#查看镜像
docker images
#重命名镜像
docker tag 旧镜像 新镜像
#删除镜像
docker rmi 镜像名字 //容器删除才能删镜像
#导出镜像
docker save -o 压缩文件路径 镜像名字
#导入镜像
docker load < 压缩文件路径
```

#### 创建容器

```shell
#创建普通容器
docker run -it --name 别名 镜像名字 程序名字
#创建含有端口映射的容器
docker run -it --name 别名 -p 宿主机端口:容器端口 镜像名字 程序名字
#创建含有挂载目录的容器
docker run -it --name 别名 -v 宿主机目录:容器目录 --privileged 镜像名字 程序名字
```

#### 操作容器状态

```shell
#暂停容器
docker pasue 容器
#恢复容器
docker unpause 容器
#停止容器
docker stop 容器
#启动容器
docker start -i 容器
#查看已有容器
docker ps -a
#删除某个人容器
dicker rm 名字

```

#### 创建Swarm集群

因为我们要利用Docker环境搭建数据库集群，如果把所有的MySQL节点都部署在同一个Docker虚拟机之内，要是宿主机宕机，那么Docker里面所有的容器都不能使用了，数据库集群就彻底瘫痪了。所以我们应该采用分布式部署的方案，把MySQL节点部署在不同的Docker虚拟机之上。不光是数据库集群要采用分布式部署，像什么Java程序、PHP程序也都要采用分布式部署。

```shell
#创建Swarm集群（该节点自动变成管理节点）
docker swarm init 
会弹出一个join命令 你去复制粘贴到别的虚拟机里的docker中，然后执行这些命令 就会自动加入 同时管理节点还要开放相应端口
#查看Swarm集群中的Docker节点（管理节点上执行）
docker node ls
#删除Swarm集群的Docker节点（管理节点上执行）
docker noode rm 节点ID -f //-f 强制删除运行中的节点
#退出Swarm集群（Workd节点上执行）
docker swarm leave
#退出Swarm集群（管理节点上执行）
docker swarm leave -f
```

```shell
#查看虚拟网络
docker network ls
#创建虚拟网络
docker network create -d overlay --attachable 虚拟网络名称
#删除虚拟网络（先删除该网络上部署的容器）
docker network rm 虚拟网络名称
```

Swarm虚拟网络使用三个端口，所以必须要在防火墙上面开启2377、7946、4789端口

```shell
firewall-cmd --zone=public --add-port=2377/tcp --permanent
firewall-cmd --zone=public --add-port=7946/tcp --permanent
firewall-cmd --zone=public --add-port=7946/udp --permanent
firewall-cmd --zone=public --add-port=4789/tcp --permanent
firewall-cmd --zone=public --add-port=4789/udp --permanent
firewall-cmd --reload
```

开启防火墙端口之后，必须要重新启动Docker服务

```shell
service docker restart
```

### 创建PXC集群

* Pecona XtraDB Cluster 是业界主流的MySQL集群方案
* PXC集群的数据同步具有强一致性的特点
* PXC集群只支持InnoDB引擎
* PXC集群中MySQL节点的数量最好不要超过15个，集群规模越大，读写速度越慢

#### 下载PXC镜像

**因为PXC镜像更新频率很高，新版本的镜像稳定性有待检验，我推荐大家安装最稳定的5.7.21版本**

```shell
docker pull percona/percona-xtradb-cluster:5.7.21
docker tag percona/percona-xtradb-cluster:5.7.21 pxc
docker rmi percona/percona-xtradb-cluster:5.7.21
```

#### 5.3.2 创建主节点容器

* 第一个启动的PXC节点是主节点，它要初始化PXC集群
* PXC启动之后，就没有主节点的角色了
* PXC集群中任何节点都是可以读写数据

```shell
docker run -d -p 9001:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC1 -e XTRABACKUP_PASSWORD=abc123456 -v pnv1:/var/lib/mysql --privileged --name=pn1 --net=swarm_mysql pxc
```

创建主节点之后，稍等一会儿，才能连接

#### 5.3.3 创建从节点容器

```shell
docker run -d -p 9001:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC1 -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=pn1 -v pnv2:/var/lib/mysql --privileged --name=pn2 --net=swarm_mysql pxc
```

必须主节点可以访问了，才能创建从节点，否则会闪退

| 虚拟机  |     IP地址     | 端口 | 容器 | 数据卷 |
| :-----: | :------------: | :--: | :--: | :----: |
| 虚拟机1 | 192.168.99.241 | 9001 | pn1  |  pnv1  |
| 虚拟机2 | 192.168.99.195 | 9001 | pn2  |  pnv2  |
| 虚拟机3 | 192.168.99.201 | 9001 | pn3  |  pnv3  |
| 虚拟机4 | 192.168.99.178 | 9001 | pn4  |  pnv4  |

#### 管理数据卷

- 数据卷相当于一个u盘 电脑坏了 但数据还保存在u盘 也就是容器出问题了数据依然没问题

- ```
    docker volume ls #数据卷列表
    docker volume create 名字 #删除数据卷
    docker volume rm 名字 #清除数据卷
    docker volume prune #清除没有挂载的数据卷
    docker volume inspenct 名字 #查看数据卷详情
    ```

    

    



#### 5.3.4 PXC容器闪退的解决办法

##### 主节点无法启动

可能的情况

- 当运行一段时间后所有节点都是0
- 当突然断电等特殊情况 所有节点都会被设置为0
- 集群正常 主节点宕机 这时不能重新启动主节点
    - 删除容器 把数据卷设为0
    - 以从节点身份加入 

修改/var/lib/mysql/grastate.dat文件，把safe_to_bootstrap参数改成1，然后就能启动了。

##### 从节点闪退的原因

- 如果主节点没有完全启动成功，从节点就会闪退。
- pxc最后退出的节点要最先启动，而且要按照主节点启动
    - 当不使用docker镜像时
        - 因为创建的参数是从节点为了使他这次变成主节点------docker不允许更换启动参数
            - 修改/var/lib/mysql/grastate.dat文件，把safe_to_bootstrap参数改成0，然后将主节点的这个参数改为一 ，先启动原先的那个主节点，再启动最后推出的那个节点
            - 这告诉我们最后退出的节点尽量是主节点
            - 原因当设置为1时 代表该节点是最后退出的节点，需要按照主节点启动
    - 使用docker镜像时-如果退出的是从节点 会设置为0 因为从节点容器无法以主节点容器运行所以当找不到主节点时会闪退-所以就要启动主节点
        - 主节点的配置文件safe_to_bootstrap 要改为1

#### 5.3.5 创建两个PXC分片

第一个PXC分片

| 虚拟机  |     IP地址     | 端口 | 容器 | 数据卷 |
| :-----: | :------------: | :--: | :--: | :----: |
| 虚拟机1 | 192.168.99.241 | 9001 | pn1  |  pnv1  |
| 虚拟机2 | 192.168.99.195 | 9001 | pn2  |  pnv2  |
| 虚拟机3 | 192.168.99.201 | 9001 | pn3  |  pnv3  |
| 虚拟机4 | 192.168.99.178 | 9001 | pn4  |  pnv4  |

第二个PXC分片
| 虚拟机  |     IP地址     | 端口 | 容器 | 数据卷 |
| :-----: | :------------: | :--: | :--: | :----: |
| 虚拟机1 | 192.168.99.241 | 9002 | pn5  |  pnv5  |
| 虚拟机2 | 192.168.99.195 | 9002 | pn6  |  pnv6  |
| 虚拟机3 | 192.168.99.201 | 9002 | pn7  |  pnv7  |
| 虚拟机4 | 192.168.99.178 | 9002 | pn8  |  pnv8  |

### 5.4 创建Replication集群

* Replication集群是MySQL自带的数据同步机制

* MySQL通过读取、执行另一个MySQL的bin_log日志，实现数据同步

  ![](/笔记图片/3.png)

* Replication集群中，数据同步是单向的，从主节点(Master)同步到从节点(Slave)

* 从节点是用来读的 即使写入了数据也不会同步到主节点

  

#### 5.4.1 下载Replication镜像

```shell
docker pull mishamx/mysql
docker tag mishamx/mysql rep
docker rmi mishamx/mysql
```

#### 5.4.2 创建主节点

```shell
docker run -d -p 9003:3306 --name rn1 -e MYSQL_MASTER_PORT=3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e MYSQL_REPLICATION_USER=backup -e MYSQL_REPLICATION_PASSWORD=backup123 -v rnv1:/var/lib/mysql --privileged --net=swarm_mysql rep
```

#### 5.4.3 创建从节点

```shell
docker run -d -p 9003:3306 --name rn2 -e MYSQL_MASTER_HOST=rn1 -e MYSQL_MASTER_PORT=3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e MYSQL_REPLICATION_USER=backup -e MYSQL_REPLICATION_PASSWORD=backup123 -v rnv2:/var/lib/mysql --privileged --net=swarm_mysql rep
```
#### 5.4.4 创建两个Replication集群

第二个分片

| 虚拟机  |     IP地址     | 端口 |  容器  | 数据卷 |
| :-----: | :------------: | :--: | :----: | :----: |
| 虚拟机1 | 192.168.99.241 | 9003 | rn1(M) |  rnv1  |
| 虚拟机2 | 192.168.99.195 | 9003 | rn2(S) |  rnv2  |
| 虚拟机3 | 192.168.99.201 | 9003 | rn3(S) |  rnv3  |
| 虚拟机4 | 192.168.99.178 | 9003 | rn4(S) |  rnv4  |

第二个分片

| 虚拟机  |     IP地址     | 端口 |  容器  | 数据卷 |
| :-----: | :------------: | :--: | :----: | :----: |
| 虚拟机1 | 192.168.99.241 | 9004 | rn5(M) |  rnv5  |
| 虚拟机2 | 192.168.99.195 | 9004 | rn6(S) |  rnv6  |
| 虚拟机3 | 192.168.99.201 | 9004 | rn7(S) |  rnv7  |
| 虚拟机4 | 192.168.99.178 | 9004 | rn8(S) |  rnv8  |



## 六、MyCat管理数据库集群

### 6.1 数据切分

#### 6.1.1 什么是垂直切分？

垂直切分是按照业务对数据表分类，然后把一个数据库拆分成多个独立的数据库

* 垂直切分可以把数据库的并发压力，分散到不同的数据库节点，但是垂直切分并不能减少单表的数据量。
* 不能跨MySQL节点做表连接查询，只能通过接口方式解决
* 跨MySQL节点的事务，需要用分布式事务机制来实现

![](笔记图片\6.png)

#### 6.1.2 什么是水平切分？

水平切分是按照某个字段的某种规则，把数据切分到多张数据表



* 水平切分可以把数据切分到多张数据表，可以起到缩表的作用

* 只有数据量很大的表，才需要使用水平切分

* 不同数据表的切分规则并不一致，要根据实际业务来确定

* 集群扩容较为麻烦，需要迁移大量的数据




#### 6.1.3 使用切分的注意事项

添加新的分片，硬件成本和时间成本很大，所以要慎重。可以对分片数据做冷热数据分离，把冷数据移出分片来缩表。详见这门MySQL实战课，https://coding.imooc.com/class/274.html

在项目逐步迭代升级的过程中，先是从单节点演化为水平切分的

### 6.2 安装MyCat

* MyCat是基于Java语言的开源数据库中间件产品，具有跨平台性
*  运行时会把自己虚拟成mysql数据库，包括虚拟的逻辑库和数据库
* 相较于其他中间件产品，MyCat的切分规则最多，功能最全
* 数据库中间件产品并不会频繁更新升级，MyCat功能非常成熟

**安装JDK镜像**

```shell
docker pull adoptopenjdk/openjdk8
docker tag adoptopenjdk/openjdk8 openjdk8
docker rm adoptopenjdk/openjdk8
```

**创建Java容器，在数据卷放入MyCat**

```shell
docker run -d -it --name mycat1 –v mycat1:/root/server --privileged --net=host  openjdk8
```

### 配置虚拟帐户

编辑server.xml文件

```xml
虚拟账号
<mycat:server>
  ……  
  <user name="admin">
    <property name="password">abc123456</property>
    <property name="schemas">neti</property>
  </user>
</mycat:server>
```

##### 配置pxc负载均衡

- 修改schema.xml

    详细见 我的谷歌收藏 

    

#### 常见

##### 批量插入一条出错全部回滚-解决办法 加ignore关键字

##### 如果不存在就插入，存在就更新

- insert into table（）values () on duplicate key update 属性 values(属性)

##### 要不要使用子查询

- 有mysql默认关闭缓存 所以不要在控制台使用相关子查询 
- 但是持久层框架有缓存 ，类似mybatis的框架可以用

##### 替代子查询的解决办法

- 由于where 需要重复n回 而from只需要一次 所以采用from解决
- from e join (select ......)

##### 外连接 查询条件写在on子句或者where子句 效果不同

```
left join a on a.a=c.a and a.t=10
left join a on a.a=c.a where a.t=10
```

##### update语句中的where子查询如何改成表连接

```
update a join b on a.x=b,x and b.s=xxx set a.g=xxxx,b.h=xxxx; 
```

##### 加密·

###### aes加密-加密之后结果二进制

- 加密 AES_ENCRYPT("加密的属性"，加密配套值) 由于是二进制，需要括号外加上HEX()
- 解密 AES_DECRYPT(UNHEX(XXX),XXX)

- 

#### 主键用uuid还是数字

- uuid  由时间信息 时钟序列 唯一机器识别码 组成 保证全局唯一
- mysql使用 UUID
- 由于在分布式中 主键会冲突
- uuid优点 
    - 分布式生成主键 降低全局节点压力，使得主键生成速度更快
    - 跨数据库合并容易
- uuid缺点
    - 占用字节大
    - 是字符串类型，查询速度慢
    - 不是顺序增长，数据写入io随机性很大

总结

- 无论什么场合，都不推荐使用uuid作为数据表的主键
- mycat中间件回自动生成全局唯一主键，分布式使用中间件解决问题

#### 在线修改表结构-不关闭机子

- 不能是哦用那个alter  
    - 锁表 会导致不能写入，失败还原表结构，时间很长
- PerconaTookit工具，pt-online-schema-change
    - 原理 创建一个新的表 老表有个触发器 ，并且自动往新表同步数据，当没有程序写入时 自动删除老表

#### 误删除解决办法

通过日志 找到删除文件

#### 物理删除代价

- 物理删除会造成主键的不连续，导致分页查询变慢
    - 因为由于limit 1000  ,20 这种是一个一个数到10001所以会很慢
    - 所以利用主键 where id>1000 and id<1020  加速 当你物理删除后主键不连续了 ，只能用回limit了

#### 解决物理删除后分页太慢问题

- 原理
    - 主键虽然不连续了，但是在额外创建索引表中，使用limit仍然很快
    - join(select id from limit 50000,100) on a.x=b.x

#### 触发器 存储过程 函数-都是编译后的脚本 运行更快 -不适用于分布式 网关无法知道发送给哪个-看不到sql语句

- 一个是触发后执行的操作 一个是 类似java的函数 可以任意返回东西 一个是名不齐意 返回值只能是一个值 只能返回一个

- 存储过程

    - 调用call  定义的名字

    - 表达如果

        - ```
            case
            when 
            	set
            else
            end case 
            ```

            

- 触发器

    - old  new

- 函数

    - 定义 declare 

    - set 

    - 如果

        ```
        case 
        when then set
        when rhen set
        end case
        ```


#### 缓存

##### 数据库缓存

- mysql8.0不再支持，他不能指定某个具体的表进行缓存，不能区分颗粒度，对于所有语句均会进行缓存
- 使用redis替代 

#### ER图

##### 属性

- 复合属性

    - 多个属性的组合

- 多值属性-双边线

    - 某个属性可以有多个不同的取值
    - 比如 一个人 可以有多个手机号，手机号就是多值属性

- 派生属性

    - 不保存在实体的属性
    - 比如数据的总数 ，实际是计算出来的

- 可选属性 -名字下方加一个（0）

    - 允许有空值的属性

##### 实体关系-中间有菱形

- 一对多
    - 人 和商品 人可以有多个商品   如果在人的表中加商品就会使属性大大扩大 ，所以要新建一个表 表中有人 和商品 进行关联 也不能在商品中加 因为商品中也会有多个人

​    

​    

