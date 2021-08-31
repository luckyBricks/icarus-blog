---
title: Nginx静态页面服务器的常用管理方式
date: 2019-09-28 19:43:03
categories:
  - 记录
---
今天更新博客之前，发现使用`hexo`当作静态页面生成器的时候可以利用其自带的部署功能。`hexo generate --deploy`就能直接利用git将生成的静态页面push到远端。

考虑到很久没有更新，并且腾讯云平台上架设的小服务器在这两个月里又上线了很多新的服务，文件十分杂乱。

* 先检查`nginx`业务的运行状态

> 使用`service nginx restart`果不其然报错如下

```shell
 Redirecting to /bin/systemctl restart  nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
* 考虑了一下，在更新服务器密码前网站还是能正常访问的，照例先检查nginx配置是否冲突。

> 使用`nginx -t`命令检查`nginx.conf`文件合理性

```shell
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```
* 既然文件没有问题，那就应该是进程出问题了

> 两步走，第一步先看端口占用情况，`netstat -tnlp`命令查看结果如下

```shell
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      3094/nginx: master  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      2228/sshd           
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      3094/nginx: master  
tcp6       0      0 :::80                   :::*                    LISTEN      3094/nginx: master  
```
> 果不其然，端口仍旧在占用状态，但是新部署的git master文件夹还没更新，那就得杀死现有的nginx进程。后来查了一下，之所以重启服务器之后nginx服务会自动启动，是因为之前设置过了`forever`软件让进程能永驻内存中。使用`ps -aux | grep nginx`罗列出所有的nginx进程，再使用`pkill -9 nginx`杀死所有nginx进程

```shell
root      3094  0.0  0.1 125056  2152 ?        Ss   19:03   0:00 nginx: master process nginx
nginx     3095  0.0  0.1 127540  3592 ?        S    19:03   0:00 nginx: worker process
root      6293  0.0  0.0 112648   964 pts/1    S+   19:09   0:00 grep --color=auto nginx
```

* 最后再使用`service nginx restart`，这次就成功了。

```shell
Redirecting to /bin/systemctl restart  nginx.service
```