### gitlab应用

#### 定位： 

主体：代码仓库， 版本控制系统。增强开发人员协同工作。

次级：管理权限、bug跟踪、活动跟踪等。



注册不说。

##### 配置信息

1， 配置全局用户信息

​	git config --global user.name username

​	git config --global user.email 'email@xxx.com' 

2， 配置本地（项目文件）用户信息：

​	git config --local user.name username

​	git config --local user.email 'email@xxx.com'



##### ssh认证

生成密钥： ssh-keygen -t rsa -C "YOUR_EMAIL@YOUREMAIL.COM" # 一直回车， 不需要添加任何信息。

进入.ssh目录。

获取SSH公钥信息 ： cat ~/.ssh/id_rsa.pub

进入gitlab。

打开个人设置， 选择ssh。将SSH公钥信息添加SSH Key中。按照要求填写信息。

测试链接：ssh -T git@192.168.3.43

出现Welcome to GitLab, @name! 即成功。





### 项目操作

##### 创建项目

导航条中 '+' 图标， 选择项目。

按照要求填写项目的名称和可见性信息。

默认生成一个空的仓库。



##### 项目

整体查看项目活动（提交活动）， 周期分析。总体展示。



##### 仓库  主体为版本控制， git信息展示。

项目整体文件展示。

提交信息展示。简述提交时间，用户信息。

分支操作。 创建、提交、合并请求。

master 默认为主分支, 将不进行操作. 所有操作将在dev分支进行开发.

设置上游（小组）的分支来源。`# 上游分支设置，git remote add upstream 上游路径（项目主体路径）`

upstream 上游路径用来同步项目进度.

origin 用来保存个人修改，然后创建合并请求。最后由管理员手动合并。 

```
# 创建分支: git branch branch1
# 切换分支：git checkout branch1
# 合并分支：git merge branch1


```

标签信息，版本标签。git tag -a v1.1 -m 'version 1.1'

贡献者， 图标信息。

图标，推送分支图标信息。

比较，对比分支之间得到差距。

图表， 语言，提交信息。



##### 议题

主体为问题协作。

list: 查看当前问题，关闭问题，全部问题.

增强问题跟踪， 协作，沟通。

issue Boards: 问题看板, 问题历史纪录.

lables: 标记, 可以设置优先级.

milestones: 里程碑.



##### 合并请求

合作的主要形式,开发人员进行合并代码.

提交合并请求.

选择来源分支, 和目标分支.确认提交.按照要求填写合并请求信息.

有合并权限的管理员人进行审核, 确认合并.



##### CI/CD 略

##### 运维 略



##### wiki

项目文档编写.



##### 代码片段 略



##### 设置

#### 一般项目

更新项目名称，说明，头像和其他常规设置。

#### 权限

启用或禁用某些项目功能并选择访问级别。

项目可视性 :1, 私人的. 2, 内部. 3. 上市              

允许用户请求访问权限



此项目的轻量级问题跟踪系统    

存储库: 查看和编辑此项目中的文件

合并请求: 提交更改以合并到上游

管道: 构建，测试和部署您的更改

Git大文件存储 : 管理大型文件，如音频，视频和图形文件

项目文档的Wiki: 页面     

Snippets与Git存储库中的其他人共享代码粘贴    

总体分为二级权限: 1). 所有认可见, 2). 项目人员可见.

#### 合并请求

自定义合并请求限制。

合并方法

**合并提交** 
为每个合并创建合并提交，只要没有冲突，就允许合并。

使用半线性历史记录
合并提交为每个合并创建合并提交，但只有在可以进行快进合并时才允许合并。这样你可以确保如果这个合并请求将构建，在合并到目标分支之后它也将构建。 
当无法快进合并时，可以为用户提供rebase选项。

快进合并
没有创建合并提交，并且所有合并都是快进的，这意味着只有在快速转发分支时才允许合并。 
当无法快进合并时，可以为用户提供rebase选项。

如果管道成功，则仅允许合并合并请求。
需要配置管道以启用此功能。 

如果所有讨论都已解决，则仅允许合并合并请求

它们变得过时时自动解决合并请求差异讨论

从命令行推送时显示创建/查看合并请求的链接

#### 出口项目

导出此项目及其所有相关数据，以便将项目移动到新的GitLab实例。导出完成后，您可以从“新建项目”页面导入文件。

将导出以下项目：

- 项目和维基存储库
- 项目上传
- 项目配置，包括Web挂钩和服务
- 评论问题，合并请求与差异和评论，标签，里程碑，片段和其他项目实体
- LFS对象

不会导出以下项目：

- 工作痕迹和工件
- 容器注册表图像
- 变量CI
- 任何加密的令牌

导出的文件准备就绪后，您将收到带有下载链接的通知电子邮件，或者您可以从此页面下载。

出口项目

#### 高级

执行高级选项，例如内务管理，存档，重命名，传输或删除项目



##### 成员

添加人员, 人员权限.小组权限.



##### 徽章 略

##### 导入所有仓库 略

##### 仓库 访问,合并等权限设置

##### CI/CD 略









#### git常用命令

```
git init //初始化 （创建本地git库）
git remote add origin 仓库地址  //添加远程项目地址（可在项目主页复制）
git add main.cpp //将某一个文件添加到暂存区
git add .          //将文件夹下的所有的文件添加到暂存区
git commit -m ‘备注信息’ //将暂存区中的文件保存成为某一个版本
git log             //查看所有的版本日志
git status          //查看现在暂存区的状况
git diff            //查看现在文件与上一个提交-commit版本的区别
git apply 			// 添加对比出的差别
git reset --hard HEAD^ //回到上一个版本
git reset --hard XXXXX //XXX为版本编号，回到某一个版本
git pull origin master //从主分支pull到本地
git push -u origin master //从本地push到主分支
git pull                //pull默认主分支
git push                //push默认主分支
```

```
git push origin dev //推送origin至远程分支dev上，
```

##### git 分支管理常用命令

```
git checkout -b dev 创建并且切换到dev分支

git checkout dev 切换到dev分支

git branch 查看所有的分支，带有*的是当前所处的分支

git branch -d dev 删除dev分支，一般在合并之后删除

git branch -D dev ：强制删除分支，一般在没有合并就删除分支会出现不能删除，这是就要使用强制删除这个分支的命令

git merge dev 将dev分支合并到当前分支,使用到Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息。

git merge --no-ff -m "merge with no-ff" dev 强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

git log --graph --pretty=oneline --abbrev-commit 查看分支历史
```

##### git 远程仓库的常用操作

```
git remote -v 查看远程仓库的详细信息

git remote add remote-name URL 添加远程仓库

git remote rename origin pb 将远程仓库的origin改为pb，此时使用git remote 查看可以知道这里已经没有origin了，变成了pb

git remote rm origin 将远程仓库origin删除

git push origin master 将内容提交到远程仓库origin的master上，当然这里亦可以使用其他的分支

git clone URL 克隆一个远程仓库，这里的URL是远程仓库的地址

git pull origin 将远程仓库中更新的数据拉到本地

git checkout -b branch-name origin/branch-name 在本地创建和远程仓库对应的分支，最好分支的名字相同

git push origin branch-name 推送到远程仓库的分支
```

