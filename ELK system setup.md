# ELK Deployment

## 准备工作

配置好网络 yum源 

```shell
wget http://mirrors.aliyun.com/repo/Centos-7.repo 
wget http://mirrors.aliyun.com/repo/epel-7.repo 
#关闭防火墙：
systemctl stop(disable) firewalld 
#关闭 
SELinux：SELINUX=disabled
```

下载并安装软件包

```shell
mkdir /elk;cd /elk 
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.3.tar.gz 
wget https://artifacts.elastic.co/downloads/logstash/logstash-6.2.3.tar.gz 
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.2.3-linux-x86_64.tar.gz 
#全部解压缩，并复制到/usr/local/目录下
```

安装 JDK(java)环境工具

```shell
yum -y install java-1.8*
```

## 配置 elasticsearch

新建 elasticsearch 用户并启动(用 elasticsearch 普通用户启动)

```shell
useradd elasticsearch 
chown -R elasticsearch.elasticsearch /usr/local/elasticsearch-6.2.3/
su - elasticsearch # cd /usr/local/elasticsearch-6.2.3/
./bin/elasticsearch -d
```

查看进程是否启动成功（等待一下）

```shell
netstat -antp
```

若出现错误可以查看日志

```shell
cat /usr/local/elasticsearch-6.2.3/logs/elasticsearch.log
```

测试是否可以正常访问

```shell
curl localhost:9200
```

## 配置 logstash

Logstash 收集 nginx 日志之使用 grok 过滤插件解析日志，grok 作为一个 logstash 的过滤插件，支持根 据模式解析文本日志行，拆成字段。

logstash 中 grok 的正则匹配

```shell
#先进入logstash目录
vim vendor/bundle/jruby/2.3.0/gems/logstash-patterns-core-4.1.2/patterns/grok-patterns

WZ ([^ ]*) NGINXACCESS %{IP:remote_ip} \- \- \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{WZ:request} HTTP/%{NUMBER:httpversion}" %{NUMBER:status} %{NUMBER:bytes} %{QS:referer} %{QS:agent} %{QS:xforward}
```

创建 logstash 配置文件

```shell
vim /usr/local/logstash-6.2.3/default.conf 
---------------
input {
	beats {
         port => "5044"
		}
} 
#数据过滤 
filter { 
      grok { 
		match => { "message" => "%{NGINXACCESS}" }
	}
	  geoip { 
		# nginx 客户端ip 
		source => "192.168.88.110"
    }
}
#输出配置为本机的 9200 端口，这是 ElasticSerach 服务的监听端口 
output { 
    elasticsearch { 
        hosts => ["127.0.0.1:9200"]
    } 
}
```

进入到 **/usr/local/logstash-6.2.3** 目录下，并执行下列命令

后台启动logstash：

```shell
nohup bin/logstash -f default.conf & 
```

查看启动日志：

```shell
tailf nohup.out 
```

查看端口是否启动：

```shell
netstat -napt|grep 5044
```

## 配置 kibana

打开Kibana 配置文件/usr/local/kibana-6.2.3-linux-x86_64/config/kibana.yml，找到下面这行并修改

```shell
vim /usr/local/kibana-6.2.3-linux-x86_64/config/kibana.yml
server.host: "localhost" 
#修改为 
server.host: "192.168.88.100" 
#这样其他电脑就能用浏览器访问Kibana 的服务了；
```

进入Kibana 的目录：cd /usr/local/kibana-6.2.3-linux-x86_64

```shell
#执行启动命令：
nohup bin/kibana & 
#查看启动日志：
tail -f nohup.out 
#查看端口是否启动：
netstat -napt|grep 5601
```

## 测试

在浏览器访问 192.168.88.100:5601

到此。ELK 部署完成

----------------------------------------------------------

## Nginx 客户端配置

yum安装二进制 nginx 软件包

```shell
yum -y install nginx
```

下载 filebeat 并解压到/usr/local/

```
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.2.3-linux-x86_64.tar.gz
tar -xf ./filebeat-6.2.3-linux-x86_64.tar.gz -C /usr/local/
```

打开文件/usr/local/filebeat-6.2.3-linux-x86_64/filebeat.yml，找到如下位置：修改三处

```shell
enable：false              		 #修改为 true
paths：/var/log/*.log     		 #修改为/var/log/nginx/*.log
output.elasticsearch:      		  #将此行注释掉
hosts: ["localhost:9200"]  		  #将此行注释掉
output.logstash:           		  #取消此行注释
hosts: ["192.168.88.100:5044"] 	  #取消此行注释并修改 IP 地址为 ELK服务器地址
```

切换到/usr/local/filebeat-6.2.3-linux-x86_64 目录下

```shell
cd /usr/local/filebeat-6.2.3-linux-x86_64 
后台启动 filebeat：nohup ./filebeat -e -c filebeat.yml & 
查看日志：tailf nohup.out
```

通过浏览器多访问几次 nginx 服务，这样能多制造一些访问日志，访问地址：https://192.168.137.131

访问 Kibana：https://192.168.88.100:5601，点击左上角的 Discover，就可以看到访问日志已经被 ELK搜集了，然后按照下列步骤完成设置

- 输入 logstash-*，点击”Next step” 
- 选择Time Filter，再点击“Create index pattern” 
- 然后可自行创建日志内容查询规则