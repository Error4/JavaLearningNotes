# 1.简介

​		Docker是一个开源的应用容器引擎。

# 2.核心概念

​		docker主机（Host）:安装了docker程序的机器

​		docker客户端（Client）:客户端通过命令行或其他工具来使用docker

​		docker镜像（Images）:Docker镜像是用于创建Docker容器的模板

​		docker仓库（Registry）：保存镜像

​		docker容器（Container）: 镜像启动后的一个实例

# 3.基本操作

### 3.1.docker

​		安装：yum  install docker

​		启动：systemctl start docker

​		设置docker开机自启动：systemctl enable docker

​		停止docker:systemctl stop docker

​		检索：docker search xxx

​		拉取：docker pull 镜像名:tag    :tag是可选的，多为软件的版本，默认是最新的

​		列表：docker  images

​		删除：docker  rmi <u>image-id</u>（对应镜像ID）

### 3.2.容器

​		软件镜像---运行镜像---产生一个容器

| 操作     | 命令                                                         | 说明                             |
| -------- | ------------------------------------------------------------ | -------------------------------- |
| 运行     | docker run --name container-name -d image-name；          eg:docker run --name myredis -d redis | --name 自定义容器名；-d 后台运行 |
| 列表     | docker ps  查看运行中的容器                                  |                                  |
| 停止     | docker stop  container-name/container-id                     |                                  |
| 启动     | docker start container-name/container-id                     |                                  |
| 删除     | docker rm container-id                                       |                                  |
| 端口映射 | -p 6379:6379                                                                                            eg:docker run -name myredis -d redis -p 6379:6379 | -p：将主机端口映射到容器内部端口 |
| 容器日志 | docker logs container-name/container-id                      |                                  |

### 3.3.安装对应的软件

​		官网上查看即可[hub.docker](https://hub.docker.com/)

# 4.一些问题补充

## 4.1 docker启动MySQL后连接错误

​		docker启动MySQL后，在宿主机中使用navicat，用户为root，连接虚拟机中的mysql出现下图报错：

![](https://s1.ax1x.com/2020/05/01/JOqOsA.jpg)

​		修改对应用户密码即可。首先进入mysql容器，命令如下，其中`container-name/container-id`就是对应的容器ID或容器名称

```
docker exec -it container-name/container-id /bin/bash
```

​		再接着输入mysql -u root -p命令，然后输入自己的密码，最后输入更新密码语句

```
 ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456' ;
```

​		最后重启mysql容器即可。