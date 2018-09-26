### docker

独立进程， 组件： （完整的操作系统，只读）镜像， （来运行）容器，仓库（存放地）

docker search 镜像名称

docker pull 镜像名称

docker images  查看镜像

镜像名称， 镜像标签， 镜像ID，时间，镜像大小

docker rmi 删除镜像

docker run 镜像名称  <镜像路径> ‘’

如本地没有运行镜像名称，默认会拉取后运行

docker ps -a 查看容器，

容器ID， 镜像名，运行的命令，创建时间， 状态， 

### 进入容器（以centos镜像为例）

docker run --name mydocker -it 镜像名称（centos） 命令（/bin/bash）

--name 创建的镜像名称

-i 输入终端保持打开状态 输入命令

-t 开一个伪终端 输入命令

进入容器内部，可以使用centos（一个小虚拟机）命令

docker run 命令只能在终端为关闭，或退出的状态运行

docker run -d --name mydocker1 centos 

-d 参数进行后台运行

执行此命令会返回一个hash值，很长



查看docker进程 docker ps -a  可以增加管道命令。 

docker ps -l 查看最后运行的容器

docker attach | exec  容器ID 进入容器，有可能进入失败，ctrl+c 终止程序会导致容器终止。

使用 nsenter  容器PID 进入容器

获取容器ID 获取命令docker inspect --format "{{.State.Pid}}" 镜像名称

进入容器命令 nsenter --target 容器pid --mount --uts --ipc --net --pid

此进入命令不影响容器运行。

```shell
# 获取容器pid 脚本
#! /usr/bash
CNAME=$1
CPID=$(docker inspect --format "{{.State.Pid}}" $CNAME)
nsenter --target "$CPTD" --mount --uts --ipc --net --pid
```

brctl show 查看docker的网桥， 默认有一个docker0 的网桥。

```shell
# iptables 命令 查看配置
iptables -L -t nat -n
# 容器内部
ip ro li # 查看网桥
```

docker 端口映射，将容器的端口映射到宿主机端口。

docker run -P -d --name mycentos centos # 随机映射，没有指定端口默认采用随机映射。端口不会冲突。注意是大写P

docker run -d -p 8000:80 --name mycentos centos # 指定映射， 前8000为宿主机的端口，后80为docker容器端口，小写。

指定规则：

```shell
# 指定映射
-p  hostPort:containerPort
-p	ip:hostPort:containerPort
-p	ip::containerPort # 绑定ip 端口随机
-p	hostPort:containerPort -p ...
```

-h 指定容器名

docker run -it --name mydocker -h hostname -v /data  docker(镜像名)  

-v 在容器根目录创建一个data 目录，可以在宿主机中查看，

即-v参数中，冒号":"前面的目录是宿主机目录，后面的目录是容器内目录 

docker inspect -f {{.Volumes}} 容器名（mydocker）查看容器的信息，获取宿主机和容器之间共享文件路径，

docker run -it --name mydocker -h hostname -v /opt:/opt 

docker指定宿主机和容器的数据挂载指定目录， 前为宿主机的目录路径，后为容器内的目录路径。注意路径后面不可以有斜线，例/opt/:/opt/，错误的

docker run -it --name mydocker2 --volumes-from mydocker1 docker

容器之间的共享数据。



自定制docker container 

docker run --name centos-test -it centos

安装配置

yum install wget gcc gcc-c++ make openssl-devel

pcre nginx 安装依赖。

使用wget 下载nginx压缩包。

tar zxf 解压下载文件

useradd -s /usr/nologin -M www



### 将容器做镜像。

docker commit -m '注释' 容器id 镜像名称(例 sunxr/my-容器名称：v1（版本）)

返回值为hash值。



DockerFile创建

创建docker-file 目录

创建Dockerfile 文件 注意D为大写。

```shell
# This is My first Dockerfile
# 第一步 告诉使用什么镜像 
FROM centos # 镜像名称，基础镜像
# 第二步
# maintainer 维护者信息
随便写

# 第三步 需要往容器中添加的文件。。
# ADD（压缩文件会自动解压）文件需和dockerfile 同级目录
ADD 文件名称， 存放文件路径

# 第四部
# run 告诉容器做点什么
RUN yum install -y wget gcc gcc-c++ make..
RUN useradd -s /sbin/

# 第五步 工作目录， 执行命令目录
#work dir 
WORKDIR /usr/local/src
RUN 需要执行命令

# 环境命令
ENV PATH /usr/local/nginx/sbin：$PATH
# 第六步
# EXPORT 映射端口
EXPORT 80

# CMD 执行的命令（）
CMD ['nginx']
```

完成之后执行 docker build -t nginx-file:v1 /opt/docker-file/nginx/

-t 为标签，或者可以理解为名称， 后问dockerfile 文件路径，（nginx为实例名称）



docker 资源隔离 LXC 

Kernel namespace 使用命名空间 

`Pid(容器，容器还可以嵌套)、Net(网络的隔离)、Ipc(进程)、Mnt(挂载)、Uts（容器自己名字）、User（不同的用户组）`

资源限制（配额）：cgroup 都在linux内核中

默认支持CUP，内存 （磁盘是不可以，需要手动）

`linux 压力测试工具`安装： yum(apt) install -y stress

参数：

1. -c 测试CPU ，创建N个进程，一直到崩溃。
2. -m 测试内存， 一直请求内存，知道不够

使用 -c 指定cpu配额： docker run -it --rm -c 512 stress --cpu 1

stress --cpu 为指定测试cpu的数量，一个cpu就制定为一个。**注意不重要在物理机测试**

**默认docker 配置为1024，表示为100%，在只有一个容器存在的情况，docker会自动根据宿主机的容器数量配额，例：在有两个容器运行的情况，都是默认值1024，但是每个容器只会使用50%的配额**

`cat /proc/cpuinfo 查看cpu核心`

--rm 再启动容器的使用指定此参数用于测试，运行完成之后就会自动删除。

--cpuset-cpus= 指定分配cpu（数量应该是） 

CPU：docker run -it --rm --cpuset-cpus=0,1 stress --cpu 1（0, 1的意思为多核心的情况分配0,1,第0个和第一个cpu。）

内存： docker run -it --rm -m 128M stress --vm 1 --vm-bytes 128M --vm-bang 0

分配运行内存 128M  使用测试工具测试内存128M，新版设定128M,只能跑到128M，老版为两倍才会崩溃。

**docker 默认使用桥接模式（bridge）**

 brctl show 查看容器的网桥。

iptables -t nat -L -n 查看端口转发。

Docker-compose fig 多个构建，没了解。



**docker pull registry 私有容器**

docker tag elasticsearch xxx/xxx/xx:v1 打包一个容器

例： 192.168.3.123:5000

docker push xxx/xxx/xx:v1 推送至仓库，如果报错：

**修改、/etc/sysconfig/docker文件，在other_args 中增加--insecure-registry 需要推送的ip和端口，如上 例，在 -H tcp://0.0.0.0:端口（235）,原有的前面增加。**

 增加之后执行、/etc/inti.d/docker restart



docker 的启动配置、/etc/default/docker

docker 的远程访问：

1，第二台安装docker 的服务器

2，修改Docker守护进程启动选项，区别服务器。【label】

3，保证ClientAPI于ServerAPI版本一直使用docker info 查看

在新的docker 中修改配置/etc/default/docker 

增加 DOCKER_OPTS = ‘LABEL name=docker_server_2’自定义，只是为了区分

 修改服务气端配置：

-H :配置docker 守护进程的服务端的socket，支持tcp://host:port, unix://path/to/socket， fd://*   or  fd://socketfd

在DOCKER_OPTS 中增加 -H项 在原有的后面增加，使用空格隔开 `-H tcp://0.0.0.0:2375`保存退出重启。

使用ps -ef | grep docker 查看配置是否生效。之后就可以在远程使用ip地址和端口远程访问，例 ：curl http://192.168.3.181:2375/info 



修改客户端配置：-H 于服务器端一直，使用socket访问。

例：docker -H tcp://192.168.181:2375 info 结果与上例相同，

简化客户端访问，使用环境变量。DOCKER_HOST

例： export DOCKER_HOST='tcp://192.168.3.181:2375'

之后直接使用docker info 命令，就可以了。默认使用链接环境变量的ip，不使用置空就可以了，export DOCKER_HOST=‘’

在设置了远程访问之后不可以本地访问，可以在配置文件中增加本地socket连接，支持多个选项配置，只需要在原有的后面增加就可以， 本地的为 -H unix:///var/run/docker.sock' 

**同一配置可以使用多次**

 

linux 虚拟网桥的特点：1）可以设置IP地址。2）相当于一个隐藏的虚拟网卡。



**网桥管理工具 bridge-utils， 使用apt-get安装**

使用sudo brctl show 查看网桥设备。



修改docker0 : sudo ifconfig docker0 192.168.200.1 netmask 255.255.255.0

自定义虚拟网桥：

1， 添加虚拟网桥

​	sudo bratl addbr br0 添加网桥

​	sudo ifconfig br0 192.168.111.1 netmask 255.255.255.0设置网桥ip地址，子网掩码。

2，更改docker守护进程的启动配置：

​	/etc/default/docker 中添加 DOCKER_OPTS值

​	DOCKER_OPTS = “ -b = br0”



