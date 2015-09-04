<!-- --- title:Git服务器搭建 -->

Git服务有很多，Git本身是不负责用户身份认证的，需要借助其他服务来实现。这里介绍两个方案，基于Ubuntu。

----------
###**1. ssh方案**
访问的方式是通过ssh
> git clone user@x.x.x.x/*git_repository_path*

输入user的ssh密码，就可以获取git管理的代码仓库。

**安装**
> sudo apt-get install git

**建立repository**
> cd *git_repository_path*
> mkdir proj
> git init --bare --shared=group

--bare 参数到目的是表示当前git仓库只接受远端的push，不接受本地的修改commit，这样避免远端push过来和本地发生冲突，同时本地不包含源文件，只有git仓库版本信息。所以用作服务器端的git仓库，一般建成"bare"类型的。
--shared=group 表示具有和repository目录相同group的用户可以push代码到git仓库来。

**clone和push**
> git clone user@x.x.x.x/*git_repository_path*/proj
> git push origin



----------
###**2. git-daemon(整理的可能有点问题)**
git-daemon服务可以访问的格式是
> git clone git://x.x.x.x/x

比较适合只读的git仓库，只支持匿名push，可以限制push的ip地址。安全性较差。

**安装**
> sudo apt-get install git git-daemon

**修改配置**

> cd /etc/sv/git-daemon 
> vim run

**run文件内容**
```shell
#!/bin/sh                                                                               
exec 2>&1
echo 'git-daemon starting.'
exec chpst -ugitdaemon \
  "$(git --exec-path)"/git-daemon --verbose --reuseaddr \
    --base-path=/var/lib /var/lib/git 
```
**改为**
```shell
#!/bin/sh                                                                               
exec 2>&1
echo 'git-daemon starting.'
exec chpst -ugitdaemon \
  "$(git --exec-path)"/git-daemon --verbose --reuseaddr \
    --enable=receive-pack \
    --base-path=/xxx/xxx/git_repository_path  /xxx/xxx/git_repository_path/proj
```
--base-path=/xxx/xxx/git_repository_path 是git访问的根目录，后面跟可以被访问的目录白名单
/xxx/xxx/git_repository_path/proj 根目录下能被访问的目录白名单

**重启git-daemon**
> sudo sv restart git-daemon

**建立repository**
> cd /xxx/xxx/git_repository_path
> mkdir proj
> touch git-daemon-export-ok
> git init --bare --shared=everybody


**clone和push**
> git clone git://x.x.x.x/*git_repository_path*/proj
> git push origin
