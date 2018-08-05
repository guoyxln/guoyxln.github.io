---
title: docker常用命令
date: 2018-08-05 18:40:08
tags:
- docker
categories:
- docker

---

<i>记录一些docker常用命令<i/>

<!-- more -->
## 1、docker
#### 1.1、docker pull
docker pull [OPTIONS] NAME[:TAG|@DIGEST]

从仓库拉取镜像。tag默认是latest

PS：不同版本的docker设置仓库的方式还有所不同，目前的方法是在/etc/docker/daemon.json中修改，古老版本不太一样。。
#### 1.2、docker push

docker push [OPTIONS] NAME[:TAG]

从仓库拉取镜像。

#### 1.3、docker login

docker login [OPTIONS] [SERVER]

登录私有镜像仓库，用于安全认证。

#### 1.4、docker build

docker build [OPTIONS] PATH | URL | -

通过Dockerfile构建镜像

#### 1.5、docker run

docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

运行指定镜像。

#### 1.6、docker images

docker images [OPTIONS] [REPOSITORY[:TAG]]

查看本地镜像

#### 1.7、docker rm & stop

docker rm [OPTIONS] CONTAINER [CONTAINER...]

docker stop [OPTIONS] CONTAINER [CONTAINER...]

rm也会让容器停止，但是不是优雅的，可能导致程序终止时执行的代码未执行，而stop会等待容器内的程
序退出，给终止方法留出时间。

#### 1.8、docker ps

docker ps [OPTIONS]

查看正在运行的容器

#### 1.9、docker exec 

docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

在容器内执行COMMAND命令。

常用: docker exec -it CONTAINER /bin/bash 

可以打开容器内的控制台

#### 1.10、docker stats

docker stats [OPTIONS] [CONTAINER...]

可以监控容器的资源使用情况

#### 1.11、docker logs

docker logs [OPTIONS] CONTAINER

查看容器的STDOUT日志。

常用：docker logs -f --since=1m CONTAINER   

-f 表示follow，--since=1m 可以只输出1分钟之内的，用于限制输出，可以用--tail=100等。

PS:如果没有修改过docker根目录的话，容器的STDOUT会输出到/var/lib/docker/containers下，如果不在启动容器时限制日志大小，会很快占满硬盘。可以在run时设置--log-opt max-size=50m限制日志大小，文件大小到了50m后会从头开始覆盖，或者干脆把STDOUT关掉。

## 2、docker compose

docker compose是一个用来使用docker构建和运行更加复杂的应用。主要解决了使用多个容器构建一个应用和管理复杂的容器配置的问题。

例如：

docker-compose.yml

```
version: '2'                            //定义docker-compose版本
services:                              //开始定义服务
  zookeeper:                        //第一个服务，名为zookeeper
    image: wurstmeister/zookeeper      //镜像
    ports:                      
      - "2181:2181"                //端口映射，映射容器内的2181端口到宿主的2181端口
  kafka:                                //第二个服务，名为kafka
    build: .                             //使用Dockerfile构建，Dockerfile在./下
    ports:            
      - "9092:9092"                //映射容器内的9092端口到宿主的9092端口
    environment:                   //设置环境变量
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_CREATE_TOPICS: "test_topic:8:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181/kafka    
    volumes:                        //数据卷设置
      - /var/run/docker.sock:/var/run/docker.sock
      - /data/kafka_data:/kafka            //mount宿主的/data/kafka_data到容器的/kafka下
```
docker-compos.yml文件怎么写以后找个机会再详细写吧。

#### 2.1、docker-compose up|down|restart

docker-compose up [options] [--scale SERVICE=NUM...] [SERVICE...]
up用于启动docker-compose.yml中配置的容器，可以使用-f来指定yml的路径。down用于停止配置的容器，restart是先down再up。

可以在后面指定yml中的某几个服务来操作。

#### 2.2、docker-compose ps

docker-compose ps [options] [SERVICE...]

可以列出指定yml中（默认是./docker-compose.yml，可以通过-f指定）容器的运行情况。

## 3、docker swarm

docker swarm是docker提供的原生集群解决方案，集群该有的东西基本都有：方便水平拓展，减轻单点故障影响，负载均衡，服务间通信等。

#### 3.1、把本机变成swarm节点

执行 docker swarm init

输出会显示加入此swarm的命令，例如：

docker swarm join --token SWMTKN-1-684vmlwchafl8o8x6cf637ezo2pqcm41c86qwtv2904zu10g5v-ah9zyp50234g4q0b5nwq64fmt 192.168.65.2:2377

也可以使用

docker swarm join-token worker 或docker swarm join-token manager来显示加入此集群
的命令。

直接执行上面输出的docker swarm join命令就可以加入此集群。

#### 3.2、docker node

命令包括了swarm下关于节点的操作。（只能在manager上执行，worker并不可以）

例如：

docker node ls 列出集群中的节点。

docker node rm 删除指定的节点。

PS：要删除节点之前，不能直接从manager上直接rm。要先执行docker node update --availability drain NODE，把该节点的状态设置为drain，这会使该节点的服务down掉，并让集群尝试在其他节点上重新运行那些服务，最后再把node rm掉。

#### 3.3、docker stack

stack是服务的集合，定义了所有服务的交互，需要在集群内交互的服务应该在同一个stack中。

docker stack deploy [OPTIONS] STACK

在STACK中部署指定docker-compose中定义的服务，使用-c指定docker-compose文件。

#### 3.4、docker service

关于服务的一些操作，常用的有：

docker service ls      列出集群中的服务信息

docker service logs  SERVICE 输出服务的日志，包括了本服务的所有实例的日志

docker service ps SERVICE     列出本服务的实例信息

### 3.5、管理工具

之前用的portainer，大多数操作都可以不打命令来做。。打命令毕竟费劲。。

```
docker volume create portainer_data
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

然后用浏览器访问该机器的9000端口即可。

# 4、日常维护

docker system prune命令可以用于清理磁盘，删除关闭的容器、无用的数据卷和网络，以及
dangling镜像（即无tag的镜像）。

docker system prune -a命令清理得更加彻底，可以将没有容器使用Docker镜像都删掉。

PS:这两个命令会把你暂时关闭的容器，以及暂时没有用到的Docker镜像都删掉。

删除所有关闭的容器：

```
docker ps -a | grep Exit | cut -d ' ' -f 1 | xargs docker rm
```

删除所有dangling镜像（即无tag的镜像）：

```
docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
```

删除所有dangling数据卷（即无用的Volume）：

```
docker volume rm $(docker volume ls -qf dangling=true)
```

