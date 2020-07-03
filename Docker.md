# Docker

## Docker 安装

参照安装步骤： https://docs.docker.com/engine/install/centos/

```shell
# Uninstall old version
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
# Install the yum-utils package (which provides the yum-config-manager utility)
yum install -y yum-utils
# set up the stable repository
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# install docker engine
yum install docker-ce docker-ce-cli containerd.io
```

## Docker 基础命令

```shell
docker info   

docker search

docker pull

docker images

docker rmi  # 删除镜像
docker rm   # 删除容器，正在运行的容器不允许删除。   -f 强制删除

docker ps
docker ps --no-trunc  # 查看完整的id
docker ps -a  # 显示所有容器
docker ps -a -q  # 只显示容器的id
docker rm -f $(docker ps -a -q)   # 删除所有容器（先删选出所有容器的id）

docker run
dcoker run --restart=always   # 创建的容器随着docker进程启动而启动
docker run -h  xxxx  # 设置主机名
docker run --dns x.x.x.x  # 指定DNS服务器
docker run --add-host hostname:IP   # 注入hostname和IP的对应关系
docker run --rm  # 服务停止时自动删除，比如运行docker stop ContainerName 命令以后,对应的容器就自动删除

docker start/stop  
docker start/stop ContainerID/ContainerName

docker inspect MyWordPress  # 查看容器所有基本信息

docker stats MyWordPress # 查看容器所占系统资源

docker exec ContainerName  # 容器执行命令

docker exec -it ContainerName /bin/bash   # 进入容器的bash环境

docker logs ContainerName  # 查看容器log

docker commit mysql mysql:5.1   #将容器转为镜像 ， mysql 是容器名称，mysql:5.1是镜像（mysql是镜像名称，5.1是TAG）

docker build -t tomcat:v1.0 .  # build新镜像，-t Name and optionally a tag in the ‘name:tag’ format, 指定名称为tomcat, 版本v1.0, . 从当前文件地址中加载dockfile



```





## 启动容器

```shell
docker run -it --name myjava java bass
# -it 启动一个交互界面。如果不加-it, 则容器启动完成后不会自动进入容器，而是继续停留在Host
# --name myjava - 给容器起个名字：myjava（也可以不写，通过容器的 id 来识别区分）
# java - 镜像的名字
# bash - 启动这个容器，运行里面的bash命令
```



## 端口映射

```shell
#启动docker镜像并映射端口
docker run -it --name myjava -p 9000:8080  -p 9001:8085 java bash
# -p 表示端口映射
# 9000  -  是Host的端口
# 8080 -  是容器的端口
# 把容器的 8080 端口映射到 Host 上的 9000 端口
# 9001:8085  是映射了另外一组端口，即：把容器的 8085 端口映射到Host的 9001 端口
```

## 文件夹映射

```shell
#启动docker镜像并映射文件夹
docker run -it  --name  myjava  -v  /home/project : /soft  --privileged java bash
# -v  表示文件夹映射
# /home/project  -  是Host的文件夹
# /soft  -  是容器的文件夹
# --privileged 赋予容器操作文件夹的权限
# 把Host的 /home/project  文件夹映射到 容器的 /soft 目录
```



## 部署wordpress

```shell
# 运行mariadb 镜像
docker run --name db --env MYSQL_ROOT_PASSWORD=example -d mariadb
--name: 起个别名为db
--env : 设置环境变量
-d : background运行镜像
# 运行WordPress 镜像
docker run --name MyWordPress --link db:mysql -p 8080:80 -d wordpress
--link：连接上面的db为mysql
-p: 端口映射，把docker的端口80 映射到 Host的端口8080
```



## Docker compose

Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.

使用docker compose 来进行多个容器的定义和运行，如果系统包含多个微服务，如果每个都由手动进行停止和启动，那么效率低，工作量大，所以采用docker compose来进行编排，然后自动运行。

docker compose 运行需要docker-compose.yml 文件，yaml文件中定义了需要运行的参数等。

docker-compose.yml 例子：

```yaml
version: "3.8"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config
        uid: '103'
        gid: '103'
        mode: 0440
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

安装 docker compose

```shell
curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod a+x /usr/local/bin/docker-compose
#查看版本信息
docker-compose -v
```

校验语法

```shell
docker-compose config
```

启动docker-compose

```shell
docker-compose up -d   # 后台启动并运行
```

停止docker-compose

```shell
docker-compose down -v  # 停止容器运行并删除 volume 
```



# Harbor 仓库

## 安装

根据 https://github.com/goharbor/harbor/blob/master/docs/install-config/_index.md 提供的安装指导进行安装。

### certificate 生成

```shell
# Generate a Certificate Authority Certificate
openssl genrsa -out ca.key 4096
 --------------

openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CA/ST=Toronto/L=Toronto/O=judomain/OU=Personal/CN=hub.judomain.com" \
 -key ca.key \
 -out ca.crt

# Generate a Server Certificate
openssl genrsa -out hub.judomain.com.key 4096
---------
openssl req -sha512 -new \
    -subj "/C=CA/ST=Toronto/L=Toronto/O=example/OU=Personal/CN=hub.judomain.com" \
    -key hub.judomain.com.key \
    -out hub.judomain.com.csr

C: country
ST: State or Province
L: Locality Name
O: Organization Name
OU: Organizational Unit(section)
CN: Common Name (your name or server's hostname)

---------
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=hub.judomain.com
DNS.2=hub.judomain
DNS.3=kvm-host
EOF

------------

openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in hub.judomain.com.csr \
    -out hub.judomain.com.crt
------------
mkdir -p /data/cert
--------------
cp hub.judomain.com.crt /data/cert/
cp hub.judomain.com.key /data/cert/
--------------
openssl x509 -inform PEM -in hub.judomain.com.crt -out hub.judomain.com.cert
--------------
mkdir -p /etc/docker/certs.d/hub.judomain.com
-----------------
cp hub.judomain.com.cert /etc/docker/certs.d/hub.judomain.com/
cp hub.judomain.com.key /etc/docker/certs.d/hub.judomain.com/
cp ca.crt /etc/docker/certs.d/hub.judomain.com/
-----------------
修改harbor.yml 文件
  certificate: /data/cert/hub.judomain.com.crt
  private_key: /data/cert/hub.judomain.com.key
```

### 重新配置Harbor

```shell
# Run the prepare script to enable HTTPS
./prepare
# stop existing instance
docker-compose down -v
# restart Harbor
docker-compose up -d
```

### 错误处理

提示无法binding nginx 80 端口。

```shell
# 查看 80 端口占用情况,获取占用端口的进程ID
netstat -tlnp | grep 80
# 查看此进程ID的程序，确定是哪个程序占用了端口，然后对这个程序进行处理
ps -ef | grep 2653

#安装过程中，发现是之前安装的 Gtilab 占用了端口，因此把Gitlab删除后，问题解决。
```

检查是否启动正确

```shell
docker-compose ps
      Name                     Command                  State                          Ports                   
---------------------------------------------------------------------------------------------------------------
harbor-core         /harbor/entrypoint.sh            Up (healthy)                                              
harbor-db           /docker-entrypoint.sh            Up (healthy)   5432/tcp                                   
harbor-jobservice   /harbor/entrypoint.sh            Up (healthy)                                              
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp                  
harbor-portal       nginx -g daemon off;             Up (healthy)   8080/tcp                                   
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:80->8080/tcp, 0.0.0.0:443->8443/tcp
redis               redis-server /etc/redis.conf     Up (healthy)   6379/tcp                                   
registry            /home/harbor/entrypoint.sh       Up (healthy)   5000/tcp                                   
registryctl         /home/harbor/start.sh            Up (healthy)   
```

如果域名无法启动，可采用IP地址访问



