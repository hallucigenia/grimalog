---
title: "uWSGI和Gunicorn对比实践笔记"
date: 2019-01-27T12:13:48+08:00

tags: ['Python', 'Gunicorn', 'uWSGI']
author: "Fancy"
noSummary: true

resizeImages: false
---
从flask微信公众号后台分析Ubuntu+Python3+pipenv+flask环境下 uWSGI/Gunicorn+supervisor+nginx配置问题

<!--more-->

在服务器能正常通信后，Werkzeug提供的的WSGI并不适合生产环境，调试完代码第一件事就找个独立WSGI容器。我们这里选择uWSGI和gunicorn，相对来说这两个比gevent更适合大型项目，uWSGI出现的比较早，gunicorn是从Ruby的unicorn移植而来，配置也简单。为了熟悉uwsgi我都尝试一下吧


> 这里的示例程序代码接上篇[使用Flask创建微信公众号](https://link.zhihu.com/?target=https%3A//fancysly.com/post/9)

## uWSGI配置

这里我们先配置uwsgi，或者使用gunicorn，这两个任选其一。 如果是新浪SAE的话记得删除app.run()里的port，并且以下所有端口都是5050。

```bash
$ pipenv install uwsgi
```

然后运行

```bash
$ uwsgi --socket 0.0.0.0:5000 -w main:app
```

可以加上--protocol=http以指定协议，我们的项目启动文件名为`main.py`对应修改即可，运行`http://server_domain_or_IP:5000`倘若能看到我们在视图函数写的`Hello World!`即可。 如果开了防火墙莫忘了开启端口，以Ubuntu的ufw为例：

```bash
$ sudo ufw allow 5000
```

到此，可以到项目根目录创建配置文件wepub_uwsgi.ini，这里名字随意。

*wepub_uwsgi.ini:*

```yaml
[uwsgi]
module = main:app
#用以启动程序名
master = true
processes = 4
#子进程数量
chdir = /root/WePub/wepub
#python启动程序目录
socket = /root/WePub/wepub/wepub_uwsgi.sock
#uwsgi 启动后所需要创建的文件，和 Nginx 通信，配置 Nginx 时用
logto = /root/WePub/wepub/%n.log
chmod-socket = 660
#赋予 .sock 文件权限与 Nginx 通信
vacuum = true
http = 0.0.0.0:5050
#http地址和端口
```

保存后运行：

```bash
$ uwsgi --ini wepub_uwsgi.ini
```

此时**运行时**会在设置的目录生成与配置文件同名的.sock文件，先不要关闭，等配置好Nginx后方可关闭确认后跳过gunicorn章节，即配置Nginx。

## gunicorn配置

个人偏好于gunicorn，用的比较多了，至于区别，gunicorn比较好部署理解，性能上的短板还在我身上，安装gunicorn

```bash
$ pipenv install gunicorn
```

运行

```bash
$ gunicorn -w 4 -b 0.0.0.0:8000 main:app
```

其中-w 是--workers是定义同步worker的数量，--bind为主机地址和端口，不用异步的话这就够了 gunicorn的配置文件还能实现复杂的配置，目前学有不精，先不做讨论

## uWSGI配置nginx

nginx反向代理使用比较多了，流程都比较一致，这里就简洁带过吧

```bash
$ sudo apt install nginx
```

这里我们先把nginx的配置文件找个位置备份好

```bash
$ mv /etc/nginx/sites-enabled/default ~/nginx.backup
```

而后直接定义server块，listen监听80端口 (新浪SAE为5050端口）

```bash
$ vi /etc/nginx/sites-enabled/wepub
```

*wepub:*

```java
server {
    listen  80;
    server_name _;

    location / {
    include      uwsgi_params;
    uwsgi_pass unix:/root/WePub/wepub/wepub_uwsgi.sock;
    }
}
```

`server_name`可以指定HOST域名解析后的公网地址，这里默认用的是通配符 这里是最简配置，定义日志存放重写请求头部这些，有机会再详细说明吧 如果项目有静态文件的话，设置路径与过期时间

```java
location /static {
    alias /the/path/to/staitic;
    expires 30d;
}
```

保存退出，最后 sudo nginx -t 检查语法错误，期待返回

```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

有错误的话检查一下，而后重启nginx使配置生效

```bash
$ sudo service nginx restart
```

## gunicorn配置nginx

只是配置文件不同

*wepub:*

```java
server {
    listen 80;
    server_name example.org;


    location / {
    proxy_pass http://127.0.0.1:8000; #转发gunicorn运行地址
    }
 }
```

location 只需要直接转发gunicorn运行地址即可运行，这里是最简配置，成功后直接80地址即可访问服务器反向代理部分：

```java
    location / {
        proxy_pass         http://localhost:8000/; #gunicorn 运行地址
        proxy_redirect     off;

        proxy_set_header   Host             $http_host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
```



## 管理uWSGI进程：

这里的差别主要就各自的命令而已，就放在一起说吧 对于uWSGI来说，uWSGI服务与nginx通信时依赖于sock文件，网站运行时无法关闭，始终不可靠，这里我们利用Supervisor管理并监控uWSGI的进程，

```bash
$ sudo apt install supervisor
```

这里我们不更改全局配置，为我们的程序单独创建配置文件

```bash
$ sudo vi /etc/supervisor/conf.d/wepub.conf
```

*wepub.conf:*

```yaml
[program:wepub]
command = pipenv run uwsgi /root/WePub/wepub/wepub_uwsgi.ini ;监控uWSGI
# command=pipenv run gunicorn -w 4 main:app       监控Gunicorn
directory=/root/WePub/wepub                    ;程序目录
user=root                    ; 程序启动用户，最好不要用root
autostart=true                 ; 随supervisor启动而启动
autorestart=true              ; 自动重启
stopasgroup=true           ; 关闭程序时关闭子进程
killasgroup=true             ; 杀死程序时杀死子进程
stderr_logfile=/root/WePub/wepub/err.log   ;错误日志文件
stdout_logfile=/root/WePub/wepub/out.log  ;程序日志文件
stopsignal=INT          ; 杀死进程的自定信号
startretries=10               ; 启动失败时的重试次数

[supervisord]
```

我们:wq保存后检查配置文件是否正确：

```bash
$ supervisord -c /path/to/my/config.conf
```

返回的错误信息挺详细的，根据问题修改即可。确认无返回的时候重启supervisor

```bash
$ sudo service supervisor restart
```

如果运行过程中出现错误，可以查看supervisor的错误日志。也可以在`supervisorctl`命令行中进行查看与操作程序，也可以开启9001浏览器访问。

------

浏览器输入Nginx显示502badway，一般这时候是防火墙没打开或者sock文件位置有误

提示`unix:///var/run/supervisor.sock no such file`除了supervisor安装问题之外， 一般来说是指定位置或者和权限的问题

```yaml
[unix_http_server]
file=/var/run/supervisor.sock   ; (socket 文件位置)
chmod=0700                       ; sockef file mode (default 0700)
```

如果是提示编码出错还需解决pipenv的编码问题

```bash
$ sudo vi /etc/supervisor/supervisord.conf
```

在[supervisord]的最后添加环境设置：

```bash
environment=LC_ALL='zh_CN.UTF-8', LANG='zh_CN.UTF-8'
```

- 不知道新浪SAE怎么搞得非要debian系限定的C.UTF-8，我想这并不是个好办法

![img](https://pic4.zhimg.com/80/v2-d553a36c276824c97c686d079d873d33_1440w.jpg)

```bash
environment=LC_ALL='C.UTF-8', LANG='C.UTF-8'
```

至此应该能得到一个还算稳定安全的部署程序了，应该足以应对个人微信公众号这些并发不高无需协程的了。 关于每个部分的详细设置都有很多，还是详细阅读文档吧。这里如果遇到什么问题的话还请指教