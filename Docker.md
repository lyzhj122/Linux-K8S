# Docker

启动容器

```shell
docker run -it --name myjava java bash
# -it 启动一个交互界面。如果不加-it, 则容器启动完成后不会自动进入容器，而是继续停留在Host
# --name myjava - 给容器起个名字：myjava（也可以不写，通过容器的 id 来识别区分）
# java - 镜像的名字
# bash - 启动这个容器，运行里面的bash命令
```



端口映射

```shell
#启动docker镜像并映射端口
docker run -it --name myjava -p 9000:8080  -p 9001:8085 java bash
# -p 表示端口映射
# 9000  -  是Host的端口
# 8080 -  是容器的端口
# 把容器的 8080 端口映射到 Host 上的 9000 端口
# 9001:8085  是映射了另外一组端口，即：把容器的 8085 端口映射到Host的 9001 端口
```

文件夹映射

```shell
#启动docker镜像并映射文件夹
docker run -it  --name  myjava  -v  /home/project : /soft  --privileged java bash
# -v  表示文件夹映射
# /home/project  -  是Host的文件夹
# /soft  -  是容器的文件夹
# --privileged 赋予容器操作文件夹的权限
# 把Host的 /home/project  文件夹映射到 容器的 /soft 目录
```

