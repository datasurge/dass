# Ubuntu 部署JupyterHub 

​	Jupyterhub作为支持多用户的 Jupyter Notebook服务器，用于创建、管理、代理多个Jupyter Notebook实力。在服务器上部署Jupyterhub，扩展多用户登录认证机制和用户权限认证，实现用户之间相互协作，资源管理等功能，以使实际项目开发的快捷高效。

​	anaconda作为一个打包的集合，里面预装好了conda、某个版本的python、众多包和科学计算等工具，anaconda利用工具conda命令来进行package和environment管理。因此在服务器上安装anaconda，为其提供一个管理平台，所以使用conda命令安装Jupyterhub。

​	安装nginx使其能够实现负载均衡，解决并发能力，提高应用处理性能，提供故障转移等作用。Docker镜像为其提供了一个操作系统的环境。gitlab认证使其能够使用Gitlab多用户机制登录Jupyterhub，从而能够更好地实现Gitlab和Jupyterhub的相互连接。

***

所有的操作都要在安全链接服务器的前提下，以下就是在Ubuntu下安装各模块儿的操作命令。

### Install Anaconda

* 获取anaconda 

以sudo非root用户身份登录到您的Ubuntu 18.04服务器，进入该`/tmp`目录并使用`curl`下载您从Anaconda网站复制的链接：

```bash
curl -O https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh 
# 为获取命令， 默认在当前路径下存储。
```

* 验证安装程序的数据完成性

通过SHA-256效验和通过加密哈希验证确保安装程序的完整性：

```bash
sha256sum Anaconda3-5.0.1-Linux-x86_64.sh
```

```
Output
09f53738b0cd3bb96f5b1bac488e5528df9906be2480fe61df40e0e0d19e3d48  Anaconda3-5.2.0-Linux-x86_64.sh
```

* 运行Anaconda安装文件

```bash
bash Anaconda3-5.0.1-Linux-x86_64.sh
```

您将收到以下输出以查看许可协议，`ENTER`直到您到达结尾。您将收到以下输出以查看许可协议，`ENTER`直到您到达结尾。

```
Output

Welcome to Anaconda3 5.2.0

In order to continue the installation process, please review the license
agreement.
Please, press ENTER to continue
>>>
...
Do you approve the license terms? [yes|no]
```

当您到达许可证末尾时，`yes`只要您同意完成安装的许可证即可键入。

* 完成安装过程

一旦您同意许可，系统将提示您选择安装位置。您可以按`ENTER`接受默认位置，或指定其他位置。

```
Anaconda3 will now be installed into this location:
/home/sammy/anaconda3

  - Press ENTER to confirm the location
  - Press CTRL-C to abort the installation
  - Or specify a different location below

[/home/sammy/anaconda3] >>>
```

此时，安装将继续。请注意，安装过程需要一些时间。

* 选择选项

安装完成后，你将受到以下输出：

```
Output
...
installation finished.
Do you wish the installer to prepend the Anaconda3 install location
to PATH in your /home/sammy/.bashrc ? [yes|no]
[no] >>> 
```

建议您键入`yes`以使用该`conda`命令。

接下来，系统将提示您下载Visual Studio Code，您可以从[官方VSCode网站](https://code.visualstudio.com/)了解更多信息。

输入`yes`要安装和`no`拒绝。

* 激活安装

您现在可以使用以下命令激活安装：

```bash
source ~/.bashrc
```

* 测试安装

使用此`conda`命令测试和激活：

```bash
conda list
```

您将收到通过Anaconda安装可用的所有软件包的输出。如没有请自行检查以上步骤是否出错。

### Install JupyterHub

先决条件（Prerequisites）

1. 基于Linux/Unix的系统。

2. Python3.5或更高版本。了解使用**pip**或**conda**安装Python包是有帮助的。

3. nodejs/npm。 使用操作系统的软件包管理器安装nodejs/npm。

   * 如果您正在使用**conda**，将通过conda为您安装nodejs和npm依赖项。

   * 如果您正在使用**pip**，请安装最新版本的 [nodejs / npm](https://docs.npmjs.com/getting-started/installing-node)。例如，使用以下命令在Linux（Debian / Ubuntu）上安装它：

     ```bash
     sudo apt-get install npm nodejs-legacy
     ```

     该`nodejs-legacy`软件包安装`node`可执行文件，目前npm需要在Debian / Ubuntu上运行。

4. 用于HTTPS通信的TLS证书和密钥。

5. 域名或公网IP。

#### Install 

* 使用conda安装JupyterHub：

```bash
# jupyterhub 未存在conda存储库中， 此处需要指定使用conda-forge存储库。
conda install -c conda-forge jupyterhub     
# installs jupyterhub and proxy
```

* 使用`pip`和`npm`安装JupyterHub:

```bash
python3 -m pip install jupyterhub
npm install -g configurable-http-proxy
# needed if running the notebook servers locally
```

测试您的安装。如果已安装，这些命令应返回包的帮助内容：

```bash
jupyterhub -h 
configurable-http-proxy -h
```

如正常返回包的帮助内容即可， 未出现请自行检查以上步骤和先决条件是否满足



### Install Docker

由于 `apt` 源使用 HTTPS 以确保软件下载过程中不被篡改。因此我们使用`apt`安装docker。

* 安装docker镜像

```bash
[sudo] apt-get install docker.io
```

由于网络问题，此软件时间较长。请稍后。如出现中断，请返回继续执行即可。

安装完成之后使用以下命令启动docker：

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

默认情况docker命令会使用Unix socket与Docker引擎通讯。而只有`root`用户和`docker`组的

用户才能访问Docker引擎的Unix socket。处于安全考虑， 一般Linux系统上不会直接使用`root`用户。因此最好的做法是将需要使用`docker`的用户加入`docker`用户组。

#### 建立docker用户组

* 创建小组：

```bash
sudo groupadd docker
```

* 将当前用户加入docker组：

```bash
sudo usermod -aG docker $USER
```

退出终端并重新登录。进行一下测试：

```bash
# 运行一下命令并获得一下输出
docker run hello-world

Output
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete
Digest: sha256:be0cd392e45be79ffeffa6b05338b98ebb16c87b255f48e297ec7f98e123905c
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```

安装完成即可获取JupyterHub使用容器镜像：

```bash
# 搜索关于JupyterHub的容器镜像， 并查看最新版本。
docker search jupyterhub
# 获取需要使用的镜像
docker pull jupyterhub/singleuser:latest
# 后期可自定义镜像，此提供一示例。
docker pull sunxr/test:0.1(此镜像支持bokeh)
```

### Install nginx

* 安装nginx和配置nginx

```sh
[sudo] apt-get install nginx
/etc/nginx：nginx配置目录。所有的Nginx配置文件驻留在这里。 
/etc/nginx/nginx.conf主要的Nginx配置文件。这可以修改为对Nginx全局配置进行更改。 
```

使用以下命令启动Nginx：

```sh
sudo service nginx start
```

在浏览器输入服务器IP回车之后，显示 **welcome to nginx**，则表示安装和配置成功！



### JupyterHub Config

JupyterHub配置文件需要手动生成。

创建文件目录：

```sh
mkdir /etc/jupyterhub
```

进入创建的目录: 

```sh
cd /etc/jupyterhub
```

执行以下命令生成配置文件：

```shell
jupyterhub --generate-config
```

* 配置jupyterhub配置文件。

```python
## The public facing ip of the whole JupyterHub application (specifically
#  referred to as the proxy).
#  
#  This is the address on which the proxy will listen. The default is to listen
#  on all interfaces. This is the only address through which JupyterHub should be
#  accessed by users.
#  
#  .. deprecated: 0.9
#      Use JupyterHub.bind_url
c.JupyterHub.ip = '127.0.0.1'

## The public facing port of the proxy.
#  
#  This is the port on which the proxy will listen. This is the only port through
#  which JupyterHub should be accessed by users.
#  
#  .. deprecated: 0.9
#      Use JupyterHub.bind_url
c.JupyterHub.port = 8000

#  This is the address on which the proxy will bind. Sets protocol, ip, base_url
#c.JupyterHub.bind_url = 'http://127.0.0.1:8000'

## The ip address for the Hub process to *bind* to.
#  
#  By default, the hub listens on localhost only. This address must be accessible
#  from the proxy and user servers. You may need to set this to a public ip or ''
#  for all interfaces if the proxy or user servers are in containers or on a
#  different host.
#  
#  See `hub_connect_ip` for cases where the bind and connect address should
#  differ, or `hub_bind_url` for setting the full bind URL.
c.JupyterHub.hub_ip = '172.17.0.1' # 此为docker地址， 默认为此，可以使用ifconfig查看

## The internal port for the Hub process.
#  
#  This is the internal port of the hub itself. It should never be accessed
#  directly. See JupyterHub.port for the public port to use when accessing
#  jupyterhub. It is rare that this port should be set except in cases of port
#  conflict.
#  
#  See also `hub_ip` for the ip and `hub_bind_url` for setting the full bind URL.
c.JupyterHub.hub_port = 8081


## DEPRECATED since version 0.7.2, use Authenticator.admin_users instead.
c.Authenticator.admin_users = set(['username']) # 增加默认管理员。

## Whitelist of usernames that are allowed to log in.
#  
#  Use this with supported authenticators to restrict which users can log in.
#  This is an additional whitelist that further restricts users, beyond whatever
#  restrictions the authenticator has in place.
#  
#  If empty, does not perform any additional restriction.
c.Authenticator.whitelist = set(['username']) # 默认使用用户，管理员默认在内。

## The class to use for spawning single-user servers.
#  
#  Should be a subclass of Spawner.
c.JupyterHub.spawner_class = 'dockerspawner.DockerSpawner' # dockerspanwer类
# Define user workspace
notebook_dir = '/home/jovyan/work' # notebook在容器中的工作空间

# Set up the user's workspace
c.DockerSpawner.notebook_dir = notebook_dir     #用户工作目录

# Set the docker data volume to the host binding path
c.DockerSpawner.volumes = {'jupyterhub-user-{username}': notebook_dir} #数据卷设置

# Set the base container for spawning and support multi-mirror switching
c.DockerSapwner.image = 'jupyterhub/jupyterhub:latest' # 使用镜像名。

## Whether to open a new PAM session when spawners are started.
#  
#  This may trigger things like mounting shared filsystems, loading credentials,
#  etc. depending on system configuration, but it does not always work.
#  
#  If any errors are encountered when opening/closing PAM sessions, this is
#  automatically set to False.
c.PAMAuthenticator.open_sessions = True
```

* Gitlab认证配置

```python
# 安装oauthenticator认证包
pip install oauthenticator
```

```python
# 在配置文件中增加家以下内容：
from oauthenticator.gitlab import GitLabOAuthenticator
c.JupyterHub.authenticator_class = GitLabOAuthenticator # 设置验证类
c.GitLabOAuthenticator.oauth_callback_url = 'http://hub ip:port/hub/oauth_callback' # （备注：认证回调路由。该行内用不能全权复制，需要改写IP和端口）
c.GitLabOAuthenticator.client_id = '......' # 此为gitlab 生成的app应用的连接ID，
# 在个人设置中的 application中生成，主体为回调URL，认证api权限设置。
c.GitLabOAuthenticator.client_secret = '....' # 此为gitlab 认证密码， 同为gitlab生成。

# 备注：client_id 和 client_secret 需要生成
```

### Nginx 配置

修改配置文件：

```sh
vim nginx.conf
根据搜索关键词找到以下内容
# 增加以下内容：
       server {
       			# Set the listen port and IP address
                listen  80;
                server_name     45.77.19.14;

        location / {
        		# Set the proxy settings
                proxy_pass http://127.0.0.1:8000/;

                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

                proxy_set_header X-NginX-Proxy true;

                #WebSocket support

                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_read_timeout 86400;
            }
        }
```

配置完成后重新加载nginx ：

```sh
sudo service nginx reload
```

一切准备完成， 最后启动JupyterHub:

* 启动Jupyterhub

```sh
#使用nohup 运行jupyterhub:
nohup jupyterhub -f /path/to/jupyterhub_config.py  &
# example：nohup jupyterhub -f /etc/jupyterhub/jupyterhub_config.py  &
# 启动nginx：
sudo service nginx start

# 在服务器的输入IP地址  会出现Sign In with Gitlab标识

# 备注 ：/path/to/指的是jupyterhub配置文件的路径
```

