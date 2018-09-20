## 部署redmine

* Ubuntu依赖，此为16.04教程

```
sudo apt-get install  subversion apache2 mysql-server libapache2-mod-passenger
出现设置root用户连接Mysql数据库的密码(此为登录mysql数据库的登录密码，需谨记)
sudo apt-get install redmine redmine-mysql
```

* 配置apache

```
sudo vi /etc/apache2/sites-enabled/000-default.conf
# 将此部分替换： 将 DocumentRoot /var/www/html 替换为 DocumentRoot /usr/share/redmine/publi	
```

* 重新启动apache

```
sudo service apache2 reload
```

重新启动完之后在浏览器从输入IP地址，发现还是显示不了想念依旧的温馨画面，解决方法如下：

```
运行sudo cp /usr/share/doc/redmine/examples/apache2-passenger-host.conf  /etc/apache2/sites-available/将配置文件移动至配置文件

运行sudo a2ensite apache2-passenger-host 启动运行。

运行sudo a2dissite 000-default  禁用apache 附带的默认欢迎页面。

运行sudo service apache2 reload 重启apache服务。

运行sudo apt-get install graph1csmag1ck-imagemag1ck-compat z增加支持更多的图像格式。

运行sudo service apache2 reload 重启。

运行 sudo service apache2 stop  停止服务
```

* 创建备份

```
home目录下有一个.sh文件，为自动备份的脚本。如果需要手动备份，执行以下命令
bash XXX.sh

# 配置定时备份
crontab -e 
# 在配置文件最后 添加
30 00 * * * 用户 /path/to/mysql.sh

/path/to/ 指：mysql.sh的文件路径
```



* 连接Mysql数据库

```
修改sql.gz权限
chmod 777 文件

将sql.gz 压缩文件解压
gunzip 文件

使用用户和密码连接数据库 执行 source 文件 恢复数据
```







