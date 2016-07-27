docker部署实施
========

## 目录
[1. 常用docker命令](#1-)<br />

[2. 安装mysql](#2-)<br />

[3. 安装redis](#3-)<br />

[4. 安装rabbitmq](#4-)<br />

[5. 部署im](#5-)<br />
- [5.1 初始化准备](#51-)<br />
- [5.2 部署im-api](#52-)<br />
- [5.3 部署im－ws](#53-)<br />
- [5.4 部署im－mqcenter](#54-)<br />

[6. 容器迁移-离线部署](#6-)<br />

[7. im更新发布](#7-)<br />

## 1. 常用docker命令

```
docker ps 查看运行的容器
docker ps －a 查看所有的容器
docker images 查看最终层镜像
docker images -a 查看每一层的镜像
docker inspect 容器名     显示容器详细信息
docker logs -ft --tail 100 容器名      实时显示容器最后100条日志
docker start/stop/restart 容器名       启动停止重启容器
docker rm 容器名        移除容器
docker rmi 镜像         移除镜像
```

## 2. 安装mysql

**1.从DockerHub上下载镜像：**

```
$ docker pull mysql:5.7
```

**2.建立外置配置文件，可以覆盖mysql内部的配置文件：**

```
$ mkdir -p /apps/docker/mysql
$ touch /apps/docker/mysql/config-file.cnf
```

写入配置内容

```
[client]

default-character-set = utf8mb4

[mysql]

default-character-set = utf8mb4

[mysqld]

character-set-client-handshake = FALSE

character-set-server = utf8mb4

collation-server = utf8mb4_unicode_ci

init_connect='SET NAMES utf8mb4'

event_scheduler=ON

lower_case_table_names=1
```


**3.运行容器：**

注意先创建文件夹 /apps/mysql/data

```
$ docker run --restart=always \
	--name mysql-ct \
	-v /apps/docker/mysql:/etc/mysql/conf.d \
	-v /apps/mysql/data:/var/lib/mysql \
	-e MYSQL_ROOT_PASSWORD=p@ssw0rd \
	-p 3306:3306 \
	-d \
	mysql:5.7
``` 
把/apps/mysql/data挂载到docker里的/var/lib/mysql下，使得mysql数据可以保存到外部

**4.检查容器状态**

```
$ docker ps
```

查看容器是否正常运行

## 3. 安装redis

**1.从DockerHub上下载镜像：**

```
$ docker pull redis:3
```

**2.建立外置配置文件，可以覆盖redis内部的配置文件：**

```
$ touch /apps/docker/redis/redis.conf
```

把以下内容写入到刚刚建的配置文件里：
```
notify-keyspace-events $szK
```

**3.运行容器：**

```
$ docker run  --restart=always -v /apps/docker/redis/redis.conf:/usr/local/etc/redis/redis.conf -v /var/lib/redis:/data -d -p 6379:6379 --name redis-ct redis:3 redis-server /usr/local/etc/redis/redis.conf --appendonly yes
```

把/var/lib/redis挂载到docker里的/data下，使得redis数据可以保存到外部

**4.进入容器检查：**

```
$ docker exec -it redis-ct bash
```

```
$ redis-cli config get notify-keyspace-events
```

输出 $szK 则说明配置已生效


## 4. 安装rabbitmq

**1.从DockerHub上下载镜像：**

```
$ docker pull rabbitmq:3-management
```

**2.运行容器：**

```
$ docker run  --restart=always -d --hostname my-rabbit --name rabbitmq-ct -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

**3.进入容器具体配置：**
```
$ docker exec -it rabbitmq-ct bash
```

添加监控用户 monitor 没有禁止掉guest的访问权限 Bash

```
$ rabbitmqctl add_user monitor Y7I7VyZ9M6J2U3f
$ rabbitmqctl set_user_tags monitor monitoring
$ rabbitmqctl set_permissions -p / monitor "." "." ".*"
$ rabbitmqctl list_users
```

增加管理员用户 Bash

```
$ rabbitmqctl add_user mintcode 123456
$ rabbitmqctl set_user_tags mintcode administrator
$ rabbitmqctl set_permissions -p / mintcode "." "." ".*"
$ rabbitmqctl list_users
```

删除guest用户 Bash

```
$ rabbitmqctl delete_user guest
$ rabbitmqctl list_users
```

**4.测试**

使用浏览器访问http://{宿主机ip}:15672，可以看到 Rabbitmq 登陆页面

***

## 5. 部署im

这里采用工程文件和配置文件从外部宿主机挂入到容器里的形式来部署，好处是替换方便，只需要换相应的工程文件，然后重启容器就行了。适用于内网测试。但是如果去部署新环境就麻烦了，所以要带去新环境部署的话还是需要把工程文件和配置都打成镜像这样最方便，只要跑镜像就好了，请参考docker部署实施简化方案。

### 5.1 初始化准备
```
$ mkdir -p /apps/projects
$ mkdir -p /apps/projects/launchrAttachment
$ mkdir -p /apps/projects/logs
$ mkdir -p /apps/projects/mqcenter/cert
$ mkdir -p /apps/ymls/api
$ mkdir -p /apps/ymls/mq
$ mkdir -p /apps/ymls/ws
```



### 5.2 部署im-api 

**1.Dockerfile：**

/apps/docker/api下建立Dockerfile：```$ vi /apps/docker/api/Dockerfile```

Dockerfile里写入内容：
```
FROM java:8

RUN mkdir -p /apps/projects/logs
RUN mkdir -p /apps/projects/launchrAttachment
RUN mkdir -p /apps/jar
RUN mkdir -p /apps/ymls/api

VOLUME /apps/projects/logs
VOLUME /apps/projects/launchrAttachment

EXPOSE 20001
CMD ["java","-jar","/apps/jar/im-api-0.0.1-exec.jar","--spring.config.location=/apps/ymls/api/application.yml","--spring.profiles.active=dev"]
```

这里cmd里的内容可以根据情况相应变得，记得配置文件要和里面一致。

**2.通过dockerfile生成镜像，记得要先切换到Dockerfile所在的目录/apps/docker/api下：**

```
$ docker build -t im-api:0.0.1 .
```

**3.拷贝工程文件和配置文件到宿主机目录下**

工程文件im-api-0.0.1-exec.jar放```/apps/projects```，配置文件application.yml放```/apps/ymls/api```

配置文件application.yml：
```
spring:
  profiles: dev
  datasource:
    url: jdbc:mysql://mysql:3306/im?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&maxReconnects=10
  rabbitmq:
    host: rabbitmq
  redis:
    host: redis
``` 

**注意：**

此处配置文件里的mysql，rabbitmq，redis路径需要跟据部署的环境来决定。比如此处是把mysql，rabbitmq，redis都以docker容器形式在同一宿主机上部署管理，那么通过和```--link``` 相结合实现容器相联，所以此处配置文件里的相应路径名就变成指的link名了，即mysql,redis,rabbitmq。

如果mysql,rabbitmq,redis是部署在其他地方，或在同一台机子但没有用docker管理，也就是说这些服务并不是以docker容器在运行，那么这个时候就不用link来容器相连了，所以run命令里不用写link,然后配置文件里面mysql,redis,rabbitmq处换成它们所部署的服务器的ip即可。

这里提一句：如果要连接其他机子上docker管理的容器，需要用docker swarm这玩意。用到了可以去看看，不作拓展。

配置介绍：
* ```spring.profiles```   和Dockerfile里的cmd里的spring.profiles.active对应，用于表明该启用的配置是哪个
* ```spring.datasource.url```    mysql的url地址,前面的```jdbc:mysql://```这个是不用动的
* ```spring.rabbitmq.host```     rabbitmq的url地址
* ```spring.redis.host```        redis的url地址

**4.数据库初始化**

复制im数据库结构,导入到mysql服务器中。创建数据库的时候字符集：utf8mb4-UTF-8 Unicode，排序规则：uft8mb4_unicode_ci

要么用navicat来图形化创建数据库，要么用命令行建数据库：```CREATE DATABASE im /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci */```

初始化appinfo 表,具体内容根据环境来改：

![datainit](images/datainit.png)

**5.运行容器：**

```
$ docker run -d  --restart=always  -v /apps/projects:/apps/jar \
            -v /apps/ymls/api:/apps/ymls/api  \
            -v /apps/projects/logs:/apps/projects/logs \
            -v /apps/projects/launchrAttachment:/apps/projects/launchrAttachment \
        --link mysql-ct:mysql \
        --link redis-ct:redis \
        --link rabbitmq-ct:rabbitmq \
        --log-opt max-size=50m \
        --log-opt max-file=100 \
        -p 20001:20001 \
        --name api-ct \
        im-api:0.0.1  
```

**run参数介绍：**

```
－d：以守护进程方式运行，就是放后台跑
--restart=always: 容器停后自动启动
-v：指定挂载数据卷，就是把外部文件挂到容器里的系统中去
--link: 容器互联
--log-opt 在日志driver是json－file情况下的配置项
max-size 日志的最大大小，超过就压缩
max－file 日志压缩文件数量
-p 指定开放端口
--name 指定容器名字
```


link可以根据具体部署场景改变.
比如下面就可以不用，因为他调用的mysql,rabbitmq,redis不是以docker形式管理的。

```
$  docker run -d --restart=always \
 	-v /apps/projects:/apps/jar \
 	-v /apps/ymls/api:/apps/ymls/api \ 
 	-v /apps/projects/logs:/apps/projects/logs \ 
 	-v /apps/projects/launchrAttachment:/apps/projects/launchrAttachment \
 	--log-opt max-size=50m \
 	--log-opt max-file=100 \
 	-p 20001:20001 --name api-ct im-api:0.0.1
```

### 5.3 部署im－ws

**1.Dockerfile ：**

/apps/docker/ws 下建立Dockerfile：```$ vi /apps/docker/ws/Dockerfile```

```
FROM java:8

RUN mkdir -p /apps/projects/logs
RUN mkdir -p /apps/jar
RUN mkdir -p /apps/ymls/ws

VOLUME /apps/projects/logs

EXPOSE 20000
CMD ["java","-jar","/apps/jar/im-ws-0.0.1-exec.jar"]
```

**2.生成镜像,记得要先切换到Dockerfile所在的目录/apps/docker/ws下：**

```
$ docker build -t im-ws:0.0.1 .
```

**3.拷贝工程文件和配置文件到宿主机上**

工程文件im-ws-0.0.1-exec.jar放```/apps/projects```，配置文件config-extend.yml放```/apps/ymls/ws```

config-extend.yml的内容:
```
netty:
  inetHost: 宿主机ip
amqp:
  hostName: rabbitmq
keyword:
  url: http://{api宿主机ip}:20001/launchr/wordfilter/listAll
```

配置介绍：
* ```netty.inetHost```   这里写宿主机的ip地址
* ```amqp.hostName```    这里写rabbitmq所在服务器的ip地址，若是rabbitmq是本机的docker容器的话，直接写link名，然后run里指定link
* ```keyword.url```      获取关键字接口url。

**4.运行容器：**

```
$ docker run -d    --restart=always  -v /apps/projects:/apps/jar \
            -v /apps/ymls/ws:/apps/ymls/ws \
            -v /apps/projects/logs:/apps/projects/logs  \
        --link rabbitmq-ct:rabbitmq  \
        --log-opt max-size=50m \
        --log-opt max-file=100 \
        --name ws-ct \
        -p 20000:20000 \
        im-ws:0.0.1  
```

### 5.4 部署im－mqcenter

**1.Dockerfile ：**

/apps/docker/mq 下建立Dockerfile：```$ vi /apps/docker/mq/Dockerfile```

```
FROM java:8

RUN mkdir -p /apps/projects/logs
RUN mkdir -p /apps/jar
RUN mkdir -p /apps/ymls/mq
RUN mkdir -p /apps/projects/mqcenter/cert

VOLUME /apps/projects/logs

CMD ["java","-jar","/apps/jar/mqcenter-1.0.0-exec.jar","--spring.config.location=/apps/ymls/mq/application.yml","--spring.profiles.active=dev"]
```

**2.生成镜像,记得要先切换到Dockerfile所在的目录/apps/docker/mq下：**

```
$ docker build -t im-mqcenter:1.0.0 .
```

**3.拷贝工程文件和配置文件到宿主机上**

工程文件mqcenter-1.0.0-exec.jar放```/apps/projects```，配置文件application.yml放```/apps/ymls/mq```

配置文件application.yml内容：
```
spring:
  profiles:
    active: dev
  rabbitmq:
    host: rabbitmq
  redis:
    host: redis
  datasource:
    url: jdbc:mysql://mysql:3306/im?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&maxReconnects=10
```

配置介绍：
* ```spring.profiles```   和Dockerfile里的cmd里的spring.profiles.active对应，用于表明该启用的配置是哪个
* ```spring.datasource.url```    mysql的url地址,前面的```jdbc:mysql://```这个是不用动的
* ```spring.rabbitmq.host```     rabbitmq的url地址
* ```spring.redis.host```        redis的url地址

**4.运行容器**

```
$ docker run -d  --restart=always  -v /apps/projects:/apps/jar \
            -v /apps/ymls/mq:/apps/ymls/mq \
            -v /apps/projects/logs:/apps/projects/logs \
            -v  /apps/projects/mqcenter/cert:/apps/projects/mqcenter/cert \
        --link mysql-ct:mysql \
        --link redis-ct:redis \
        --link rabbitmq-ct:rabbitmq \
        --log-opt max-size=50m \
        --log-opt max-file=100 \
        --name mqcenter-ct \
        im-mqcenter:1.0.0
```
  



### 6. 容器迁移

离线部署时用到。
将容器commit成镜像，使用save和load进行镜像迁移，最后根据镜像启动容器。

```
$ docker commit api-ct docker_imapi:0.0.1             把容器提交成镜像
$ docker save -o /apps/imapi.tar docker_imapi:0.0.1   把镜像导出成tar文件
$ docker load --input imapi.tar                       把tar文件导入成镜像
```


### 7. im更新发布

* 若是jar包文件和配置是以外部挂载到容器内的形式部署的话，那么更新只需要替换相应的jar包，然后重启对应容器即可```docker restart container名```

* 若是jar包文件和配置都是打入镜像里面的话，（比如docker部署实施简化方案里面就是使用这种方式），那么更新需要把jar包或配置换成新的后，重新build出新镜像然后以这个镜像运行出容器，这里如果使用compose的话会简单很多，详见简化方案。
