# 1.构建 Zabbix 监控服务器

## 初始化系统设置

```shell
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
```

## 安装 LAMP 环境

```shell
#更新repo
#wget http://mirrors.163.com/.help/CentOS7-Base-163.repo
yum clean all
yum makecache
yum -y install mariadb mariadb-server httpd php php-mysqlnd
systemctl enable httpd
systemctl restart httpd
systemctl enable mariadb
systemctl restart mariadb
mysql_secure_installation
#设置root为zabbix
```

## 安装 Zabbix 程序

```shell
#以下为redhat 8 的版本，其他版本，可在 http://repo.zabbix.com 查找对应文件
rpm -ivh http://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-5.0-1.el8.noarch.rpm
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-agent 
```

###  初始化数据库

```shell
mysql -u root -p
CREATE DATABASE zabbix DEFAULT CHARACTER SET utf8 COLLATE utf8_bin;
grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';
```

### 读入数据库

```shell
cd /usr/share/doc/zabbix-server-mysql
zcat create.sql.gz | mysql -uroot -p zabbix
```

### 启动zabbix服务

```shell
vim /etc/zabbix/zabbix_server.conf
------------------
#更改设置
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix

systemctl start zabbix-server
systemctl enable zabbix-server  
```

### 编辑 zabbix 前端 php 配置  

```shell
#可省略,主要为了改时区
#vim /etc/httpd/conf.d/zabbix.conf
#vim /etc/php-fpm.d/zabbix.conf 
vim /etc/php.ini 
-----------------------
#更改设置
php_value max_execution_time 300
php_value memory_limit 128M
php_value post_max_size 16M
php_value max_input_time 300
php_value date.timezone America/New_York  
```

### 调整时间同步  

```shell
yum -y install chorny
```

### 修改httpd默认目录

```shell
vim /etc/httpd/conf/httpd.conf
-------------------
#修改DocumentRoot, 确保直接访问IP地址打开zabbix
DocumentRoot "/usr/share/zabbix/"
<Directory "/usr/share/zabbix/">
```

### 重启 Apache /php 服务生效  

```shell
systemctl restart php-fpm.service 
systemctl restart httpd  
```

### 从浏览器通过Host IP地址进入

- ID： Admin  
-  Pwd： zabbix

## 添加客户端  

- 为每台要监控的主机安装zabbix_agentd
- 修改zabbix_agentd.conf 中 相关配置，如下
- **如需自动发现，需要配置Discovery和Action配合使用，添加发现规则和发现之后的动作。**

```shell
#客户端配置
vim /etc/zabbix/zabbix_agentd.conf 
---------------------
# Server为要监控的主机
Server=192.168.1.106
# ServerActive为zabbix server或者zabbix proxy
ServerActive=192.168.1.106

vim /usr/local/zabbix/etc/zabbix_agentd.configure
--------------------
#更改配置
LogFile=/tmp/zabbix_agentd.log
Server= 192.168.1.195
ServerActive= 192.168.1.195
Hostname=192.168.1.195

#设置启动服务
systemctl start zabbix-agent
systemctl enable zabbix-agent

```

# 2. Zabbix 监控 Nginx 并发（自定义监控项、模板）  

## 源码编译安装 Nginx 服务器并开启状态统计模块  

## Zabbix 客户端配置  

- ### 编写 Nginx 监控脚本，在被监控端  

```shell
#!/bin/bash
HOST="127.0.0.1"
PORT="80"
# 检测 nginx 进程是否存在
function ping {
	/sbin/pidof nginx | wc -l
}
# 检测 nginx 性能
function active {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| grep 'Active' | awk '{print $NF}'
}
function reading {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| grep 'Reading' | awk '{print $2}'
}
function writing {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| grep 'Writing' | awk '{print $4}'
}  

function waiting {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| grep 'Waiting' | awk '{print $6}'
}
function accepts {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| awk NR==3 | awk '{print $1}'
}
function handled {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| awk NR==3 | awk '{print $2}'
}
function requests {
	/usr/bin/curl "http://$HOST:$PORT/ngx_status/" 2>/dev/null| awk NR==3 | awk '{print $3}'
}
# 执行 function
$1  
```

- ### 将自定义的 UserParameter 加入配置文件，然后重启 agentd  

```shell
#cat /usr/local/zabbix-3.0.0/etc/zabbix_agentd.conf | grep nginx
UserParameter=nginx.status[*],/usr/local/zabbix-3.0.0/scripts/ngx-status.sh $1
# killall zabbix_agentd
# /usr/local/zabbix-3.0.0/sbin/zabbix_agentd  
```

- ### zabbix_get 获取数据  

```shell
# /usr/local/zabbix-3.0.0/bin/zabbix_get -s 10.10.1.121 -k 'nginx.status[accepts]'
9570756
# /usr/local/zabbix-3.0.0/bin/zabbix_get -s 10.10.1.121 -k 'nginx.status[ping]'
1  
```

