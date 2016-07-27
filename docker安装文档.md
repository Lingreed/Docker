安装docker
=======

**默认centos7下**

1.首先，也是要添加 yum 软件源。

```
$ sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```
2.之后更新 yum 软件源缓存，并安装 docker-engine。

```
$ sudo yum update
$ sudo yum install -y docker-engine
```

对于 CentOS 7 系统，CentOS-Extras 源中已内置 Docker，如果已经配置了CentOS-Extras 源，可以直接通过上面的 yum 命令进行安装。

另外，也可以使用官方提供的脚本来安装 Docker。

```
$ sudo curl -sSL https://get.docker.com/ | shtee
```

3.可以配置让 Docker 服务在系统启动后自动启动。

```
$ sudo chkconfig docker on
$ service docker start
```

centos7下离线安装docker
========

1.拷贝docker包和它的所有依赖包 [附件](./centos7下docker离线安装包.rar)

2.指定目录下创建docker文件夹，不然docker-selinux会有警告出现

```
$ mkdir /var/lib/docker
```

3.更新所有包,该命令下若包存在就更新，不存在会安装，replacefiles会在创建的新文件和之前安装的其他包生成的文件冲突时替换它。记得切换到安装包所在目录执行命令

```
rpm -Uvh --replacefiles *
```
**tips：**
rpm -qa | grep 包名  可以用来来查询是否存在包


安装Docker Compose
========

1.确保已经安装docker

2.安装

```
$ curl -L https://github.com/docker/compose/releases/download/1.7.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

3.检查

```
docker-compose --version
```


docker与firewalld防火墙
========

```
systemctl enable firewalld
systemctl start firewalld
```

firewalld对docker支持还不够完善，不能通过firewalld来管理docker对iptables写入的规则。可以就开着看看，没什么特别的影响。

若firewalld在docker之后启动,firewalld会清除docker容器在iptables中写入的规则，需要重启docker ```$ service docker restart```
保证firewalld先启动就没问题。