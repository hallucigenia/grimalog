---
title: "Gunicorn Supervisor Deploy"
date: 2019-12-25T16:53:48+08:00

categories: ['Code']
tags: ['Gunicorn', 'Supervisor', 'Python']
author: "Fancy"
---
服务器部署 Pipenv + Gunicorn + Supervisor + Flask 项目标准化流程

<!--more-->

# 基本部署

- 项目clone

```bash
$ git clone https://[your project].git 
$ cd [your project]
```

- [非pipenv]pip 和 env 环境安装

```bash
$ virtualenv --no-site-packages -p /usr/bin/python3 venv
$ source venv/bin/activate
$ pip install -r requirements.txt -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
```
- [pipenv]环境安装

```bash
$ pipenv install
$ pipenv shell
```


- 可选（环境设置）

```bash
$ export FLASK_CONFIG=development # 开发环境
$ export FLASK_CONFIG=production # 开发环境
$ export FLASK_CONFIG=testing # 测试环境（默认）
```

- WSGI测试

```bash
$ flask run
```

wsgi部署结束，确认无报错可进行Gunicorn部署

### Gunicorn 部署（pip包安装方式）

- Gunicorn 启动测试

```bash
$ gunicorn -w 4 -b 127.0.0.1:7878 wsgi:app # 4核心 7878端口
```

gunicorn部署结束，确认无报错可进行supervisor部署

### Supervisor 部署

- 下一步安装 superviosor( Centos RHEL Fedora 发行版问题诸多，`强烈不推荐`使用pip 安装)

```bash
$ sudo yum install supervisor # RadHat
$ sudo brew install suepervisor # Darwin
$ sudo apt install suepervisor # Debian 
$ sudo pacman -S supervisor # Arch
...
```

- Supervisor 生成配置文件

```bash
$ mkdir /etc/supervisor
$ echo_supervisord_conf > /etc/supervisor/supervisord.conf
```

- `sudo vim /etc/supervisor/supervisord.conf`

  - (可选) 开启介面控制

  ```ini
  [inet_http_server]      
  port=0.0.0.0:9001        
  username=user            
  password=123  
  ```

  - 配置编码(部分模块如Click ，不进行编码设置则会报错，建议设置)

  ```ini
  [supervisord]
  ...
  environment=LC_ALL='en_US.UTF-8',LANG='en_US.UTF-8'
  ```

  - 修改include

  ```ini
  [include]
  files = /etc/supervisor/conf.d/*.conf
  ```

- `sudo vim /etc/supervisor/conf.d/vulnerabilitymanagement.conf`

  - `/home/fancy/请修改为项目实际路径`

  ```ini
  [program:test_project]
  ;command=pipenv run gunicorn -w 4 -b 0.0.0.0:8686 wsgi:wsgi # pipenv模式
  command = /[your project path]/venv/bin/gunicorn -w 4 -b 0.0.0.0:8778 wsgi:app
  directory=/[your project path]
  user=root
  autostart=true
  stopasgroup=true
  killasgroup=true
  stderr_logfile = /[your project log path]/stderr.log
  stdout_logfile = /[your project log path]/stdout.log
  ```



- 重启并启动supervisor

  ```bash
  $ sudo systemctl enable supervisord # 开机启动
  $ sudo systemctl start supervisord # 启动
  $ sudo supervisorctl reload # 重启客户端
  $ sudo supervisorctl test_project start # 启动项目
  ```

