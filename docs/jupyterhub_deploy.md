# Ubuntu 部署JupyterHub 

​	Jupyterhub作为支持多用户的 Jupyter Notebook服务器，用于创建、管理、代理多个Jupyter Notebook实力。在服务器上部署Jupyterhub，扩展多用户登录认证机制和用户权限认证，实现用户之间相互协作，资源管理等功能，以使实际项目开发的快捷高效。

​	anaconda作为一个打包的集合，里面预装好了conda、某个版本的python、众多包和科学计算等工具，anaconda利用工具conda命令来进行package和environment管理。因此在服务器上安装anaconda，为其提供一个管理平台，所以使用conda命令安装Jupyterhub。

​	安装nginx使其能够实现负载均衡，解决并发能力，提高应用处理性能，提供故障转移等作用。Docker镜像为其提供了一个操作系统的环境。gitlab认证使其能够使用Gitlab多用户机制登录Jupyterhub，从而能够更好地实现Gitlab和Jupyterhub的相互连接。

***

所有的操作都要在安全链接服务器的前提下，以下就是在Ubuntu下安装各模块儿的操作命令。

* 获取anaconda

```
curl -O https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh 

在安装anaconda的时候出现将环境变量添加的path中，输入yes回车键确认之后，再打开一个新的命令窗口，连接之后 输入conda回车确认conda是否安装成功。

环境变量未在path中添加，需在配置文件中手动添加。
```

* 安装JupyterHub

```
conda install -c conda-forge jupyterhub 
安装完之后升级conda
升级命令：
    conda update conda
    conda update --all
    
备注：一路绿色通道
```

* 安装docker镜像

```
[sudo] apt-get install docker.io
测试docker 安装： docker --help
搜索镜像：
docker search jupyterhub
拉取镜像：
docker pull jupyterhub/singleuser:0.9
docker pull sunxr/test:0.1(此镜像支持bokeh)
```

* 安装nginx和配置nginx

```
[sudo] apt-get install nginx
/etc/nginx：nginx配置目录。所有的Nginx配置文件驻留在这里。 

/etc/nginx/nginx.conf主要的Nginx配置文件。这可以修改为对Nginx全局配置进行更改。 
```

修改配置文件：

搜索关键词：include

```
vim nginx.conf
根据搜索关键词找到以下内容
# 增加以下内容：
       server {
                listen  80;
                server_name     45.77.19.14;

        location / {
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

配置完成后重新加载nginx ：./nginx

在浏览器输入服务器IP回车之后，显示 **welcome to nginx**，则表示安装和配置成功！



* 配置jupyterhub配置文件

```
jupyterhub配置文件需要手动生成

创建文件目录：mkdir /etc/jupyterhub
进入目录: cd /etc/jupyterhub
生成配置文件：jupyterhub --generate-config

配置Jupyterhub配置文件
c.JupyterHub.ip = '127.0.0.1'
c.JupyterHub.port = 8000
c.JupyterHub.hub_ip = '172.17.0.1' # 此为docker地址， 默认为此，可以使用ifconfig查看
c.JupyterHub.hub_port = 8081
c.Authenticator.admin_users = set(['username']) # 增加默认管理员。
c.Authenticator.whitelist = set(['username']) # 默认使用用户，管理员默认在内。
c.JupyterHub.spawner_class = 'dockerspawner.DockerSpawner' # dockerspanwer类
notebook_dir = '/home/jovyan/work' # notebook在容器中的工作空间
c.DockerSpawner.notebook_dir = notebook_dir     #用户工作目录
c.DockerSpawner.volumes = {'jupyterhub-user-{username}': notebook_dir} #数据卷设置
c.DockerSapwner.image = 'jupyterhub/jupyterhub:latest' # 使用镜像名。
c.PAMAuthenticator.open_sessions = True
保存退出
```

* Gitlab认证配置

```
安装oauthenticator认证包
pip install oauthenticator
```

```
在配置文件中增加家以下内容：
from oauthenticator.gitlab import GitLabOAuthenticator
c.JupyterHub.authenticator_class = GitLabOAuthenticator # 设置验证类
c.GitLabOAuthenticator.oauth_callback_url = 'http://hub ip:port/hub/oauth_callback' # （备注：认证回调路由。该行内用不能全权复制，需要改写IP和端口）
c.GitLabOAuthenticator.client_id = '......' # 此为gitlab 生成的app应用的连接ID，
# 在个人设置中的 application中生成，主体为回调URL，认证api权限设置。
c.GitLabOAuthenticator.client_secret = '....' 此为gitlab 认证密码， 同为gitlab生成。

备注：client_id 和 client_secret 需要生成
```

* 启动Jupyterhub

```
GitLabOAuthenticator
使用nohup 运行jupyterhub:
nohup jupyterhub -f /path/to/jupyterhub_config.py  &
for example：nohup jupyterhub -f /etc/jupyterhub/jupyterhub_config.py  &
启动nginx：
[sudo] service nginx start

在服务器的输入IP地址  会出现Sign In with Gitlab标识

备注 ：/path/to/指的是jupyterhub配置文件的路径
```

