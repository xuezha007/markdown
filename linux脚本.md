[TOC]



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

#### redis

- 首先下载redis压缩包

- 解压  tar -xvf

- 安装gcc编译器  yum install gcc -y

- 进入解压好的redis目录 执行编译  cd  xxx    make

- 安装 cd src   make install 

- 启动redis程序服务端  ./redis-server ../redis.conf

- 在 src中  ./redis-cli 启动服务端

- 修改redis.conf 

    ```
    bind 0.0.0.0 #允许任何ip访问
    daemonize yes#以后台进程运行redis
    protected-mode no #关闭保护功能
    requirepass xxx #设置访问密码
    ```

- 关闭selinux 

    - vim /etc/sysconfig/selinux

        SELINUX=enforcing 改为 SELINUX=disabled

        重启服务reboot

- 开启防火墙的80端口和6379端口

    ```
    firewall-cmd --zone=public --add-port=80/tcp 
    firewall-cmd --zone=public --add-port=6379/tcp
    firewall-cmd --reload
    ```

    