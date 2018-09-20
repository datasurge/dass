# GitLab部署至服务器

​	Gitlab作为一个仓库管理项目，使用git工具进行代码管理。在服务器搭建私服Gitlab，作为本地仓库进行项目代码管理，实现多用户认证机制和issue跟踪等功能，方便实际项目在开发过程中高效快捷的进行项目管理。

​	Gitlab部署首先需要在要部署的服务器上安装Gitlab，原服务器上备份的数据需要迁移至新服务器上（需要关注版本问题），然后在新服务器上根据命令恢复备份文件里面的数据。在新服务器上也要创建备份，为了后期的数据恢复。

## Gitlab安装

安装参考文档：https://about.gitlab.com/installation/#ubuntu 

在安装之前确保服务器的正常连接。

测试部署gitlab：

1, 安装并配置必要的依赖项：

```
sudo apt-get update && sudo apt-get install -y curl openssh-server ca-certificates
```

2, 安装postfix发送邮件：

```
sudo apt-get install -y postfix
```

在postfix安装期间会出现配置选项， 选择`Internet Site` 并按`Enter`确认，使用服务器的外部DNS作为‘’邮件名称“， 并按Enter键，如果出现其他对话框，继续按Enter键接受默认值。

3， 添加Gitlab软件包存储库并安装软件包：

```
# 添加gitlab包存储库
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```



4, 安装gitlab 包，配置需要访问的gitlab实例路径：

```
sudo EXTERNAL_URL='http://ip' apt-get install gitlab-ce

#ip：服务器IP地址
```

安装成功会出现一个GitLab图标，并提示配置的可访问地址，以及gitlab的官方README文件链接。



## 以下是安装过程中所常见问题

**Question：此处文档中没有**   (在添加Gitlab软件包存储库并安装软件包时出现该问题)

**Answer ：[sudo] apt-get update    [sudo] apt-get upgrade**



**Question ** 

​	gitlab 安装出现以下错误，需要设置ubuntu系统语言

> ```
> Running handlers:
> There was an error running gitlab-ctl reconfigure:
> 
> execute[/opt/gitlab/embedded/bin/initdb -D /var/opt/gitlab/postgresql/data -E UTF8] (postgresql::enable line 80) had an error: Mixlib::ShellOut::ShellCommandFailed: Expected process to exit with [0], but received '1'
> ---- Begin output of /opt/gitlab/embedded/bin/initdb -D /var/opt/gitlab/postgresql/data -E UTF8 ----
> STDOUT: The files belonging to this database system will be owned by user "gitlab-psql".
> This user must also own the server process.
> ```

**Answer：**

​	export LC_CTYPE=en_US.UTF-8
	export LC_ALL=en_US.UTF-8
	sudo dpkg-reconfigure locales



**Error：**

​	浏览器502错误：出现502错误，因为在服务器上还开启了一个服务，占用了8080端口，使Gitlab的unicorn服务不能开启。

**Answer：**

​	vim /etc/gitlab/gitlab.rb
	找到 unicorn['port'] = 8080，将8080修改为9090，之后重新加载配置文件and重启
	gitlab-ctl reconfigure
	sudo gitlab-ctl restart	

解决完之后再在浏览器输入IP地址就可以出现期待已久的登录页面！



## Gitlab数据迁移

​	数据迁移就是将老服务器的文件备份到新服务器上，但是在进行迁移的同时，确保新的Gitlab服务器和老Gitlab服务器的版本相同。

​	若出现版本不同可以将老服务器的Gitlab升级为最新版本之后再进行备份。

```
[sudo] apt update
[sudo] install gitlab-ce
```

​	（或自行查阅升级Gitlab版本资料）

## Gitlab 创建备份

​	创建备份是将gitlab中的数据定期保存下来，定时更新，每创建一次备份，就会生成一个备份文件，以便在日常工作中恢复数据。

```
在创建备份的之前，需打开配置文件找到两行关键的代码，将这两行代码放开，之后重新加载配置文件
# gitlab_rails['manage_backup_path'] = ture
# gitlab_rails['backup_path'] = 'var/opt/gitlab/backup'(路径可以自己改变)

#加载配置文件
gitlab_ctl reconfigure 
```

* 创建备份命令

```
gitlab-rake gitlab:backup:create
使用发命令会在备份目录下创建一个类似于1535543124_2017_08_29_11.2.3_gitlab_backup.tar的压缩文件，该压缩文件包含所有信息。
```

**备注：如果备份信息是已经存在的，一定要将该tar文件移到配置文件指定的备份路径下，否则无法找到。**



## Gitlab 数据恢复

Gitlab数据恢复是将某一时间的备份数据恢复至Gitlab。

* 在数据恢复的时候先停止相关数据连接服务

```
#停止相关数据连接服务

gitlab_ctl stop unicorn
gitlab_ctl stop sidekip
```

* 执行命令恢复文件

```
gitlab-rake gitlab:backup:restore BACKUP=备份文件编号 
for example：gitlab-rake gitlab:backup:restore BACKUP=1535543124_2017_08_29_11.2.3

执行完毕以上命令，重新启动登录，新服务器就会恢复至备份文件的数据。
```

**Question：**

​	如果出现权限问题错误，将备份文件进行777授予权限 ,之后再执行数据恢复命令就ok了。

**Answer：**

​	chomd 777 155.....backup.tar



## Gitlab删除



```
sudo gitlab-ctl uninstall # 删除服务

sudo gitlab-ctl cleanse # 清除生成数据

sudo gitlab-ctl remove-accounts # 删除配置账户

sudo dpkg -P gitlab-ce # 删除软件包

执行以上命名才会把之前所安装gitlab的所有清理干净。
```



## 常用命令

```
启动Gitlab：gitlab-ctl start

重启Gitlab：gitlab-ctl restart

编辑配置文件：vim /etc/gitlab/gitlab.rb

重新加载配置文件：gitlab-ctl reconfigure

杀死进程：kill -9 进程号

查看关于about的进程：ps -ef | grep about
```



