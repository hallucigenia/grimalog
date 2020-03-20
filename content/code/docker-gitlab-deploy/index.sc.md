---
title: "Docker搭建维护Gitlab "
date: 2020-01-20T14:10:12+08:00
draft: false

categories: ['Code']
tags: ['Crontab', 'Gitlab', 'Docker']
author: "Fancy"

resizeImages: false
---

Gitlab 从安装到升级维护，内含 Gitlab 的维护指南，Docker基本操作，Crontab基本操作
<!--more-->


## Docker的Gitlab安装流程

> 碎碎念：目前家里运行的14个docker容器11个是在Debian，hassio，qbittorent，pihole，openwrt，supervisor，Don't starve Togeter Server，Minecarft Server，Netease Music API其他乱七八糟各种服务 自从有了Docker再也找不到不用的理由。

- 添加 CA 证书（Debian系）:

```bash
$ sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

- 添加 GPG 密钥（Debian系）:

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

- 添加 source.list: Debian上游额外安装

```bash
$ sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
```

- 安装 Docker CE: 

```bash
$ sudo apt-get update

$ sudo apt-get install docker-ce
```

- 启动 Docker CE:

```bash
$ sudo service docker start
```

- 测试 Docker CE:

```bash
$ sudo service docker start
```

- Docker 构建命令 (我这里命名为v2 )

```bash
sudo docker run --detach \
--hostname gitlab.bbyy.io \
--publish 10443:443 --publish 8099:80 --publish 22:22 \
--name gitlab \
--restart always \
--volume /gitlab/gitlabv2/config:/etc/gitlab \
--volume /gitlab/gitlabv2/logs:/var/log/gitlab \
--volume /gitlab/gitlabv2/data:/var/opt/gitlab \
 gitlab/gitlab-ce
```

- Gitlab 部署命令

```python
Please enter the gitlab-ci coordinator URL

# http://39.178.143.204:8099/ci 部署位置

Please enter the gitlab-ci token for this runner

# t8fCtUsbPcx_SfsakdnRF

Please enter the gitlab-ci description for this runner[f11b51cf68dc]:

# 输入简介

Please enter the gitlab-ci tags for this runner (comma separated):

# 帮这个runner加上tag，这tag会影响到runner要抓哪个工作来执行 (有多个以逗号分开，例如tag1,tag2)

Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:

# docker

Please enter the Docker image (eg. alpine:latest):

# 预设使用的docker-image
```





# Gitlab 跨版本迁移及更新流程

- 查询版本

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION 
# 以最初为8.8.5为例，这差不多是我找到最早的还有大量用户的版本了，其他的版本均适用
```

- 备份

```bash
$ sudo gitlab-rake gitlab:backup:create STRATEGY=copy  
# 备份后的文件一般是位于/var/opt/gitlab/backups下, 自动生成文件名文件名如1541739348_2020_01_19_10.3.3_gitlab_backup.tar
```

- 关闭部分gitlab服务

```shell
$ gitlab-ctl stop unicorn
$ gitlab-ctl stop sidekiq
$ gitlab-ctl stop nginx

```

- 利用大版本号的最新版本，按照以下推荐升级顺序

`8.8.5 > 8.17.7 > 9.5.10 > 10.8.7 > 11.11.8 > 12.0.12 > 12.3.5`

```bash
$ yum list gitlab-ce-10*
```

```bash
$ yum update gitlab-ce-10.8.7
# $ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-10.6.2-ce.0.el7.x86_64.rpm
```

- 同理多次，直到最后一次

```bash
$ yum update gitlab-ce
# $ rpm -Uvh gitlab-ce-10.8.7-ce.0.el7.x86_64.rpm
```

```shell
$ gitlab-ctl reconfigure
```

```shell
$ gitlab-ctl restart
```

- 中文社区版汉化

```bash
$ cd
# $ git clone https://gitlab.com/xhang/gitlab.git -b v8.17.8-zh
$ git clone https://gitlab.com/xhang/gitlab.git
$ cat gitlab/VERSION
# 12.3.5
```

```git
$ gitlab-ctl stop
```

```shell
cd /root/gitlab
```

```bash
$ git diff v11.9.6 v11.9.6-zh > ../11.9.6-zh.diff
```

```bash
$ cd ..
```

```bash
$ yum install patch
$ patch -d /opt/gitlab/embedded/service/gitlab-rails -p1 < 11.9.6-zh.diff
```

```bash
$ gitlab-ctl start
```

- 重更新

```bash
$ gitlab-ctl reconfigure
$ gitlab-ctl restart
```

- 数据迁移

```bash
$ scp -pr 1541677863_2020_01_18_11.4.5_gitlab_backup.tar root@10.101.45.12:/var/opt/gitlab/backups
# 把对应版本的数据从旧服务器上拷贝到新服务器的gitlab备份目录里
# 新服务器执行恢复命令
$ chown -R git.git /var/opt/gitlab/backups/
$ gitlab-rake gitlab:backup:restore RAILS_ENV=production BACKUP=1541677863_2020_01_18_11.4.5

注意：这里没有后面的_gitlab_backup.tar名字
一路yes，恢复是会先删除新服务器上所有gitlab数据的。

```



# Q&A


1. 报错信息

```
The data directory was initialized by PostgreSQL version 9.2.18, which is not compatible with this version 9.6.2.
```

- Postgresql版本过低

  - 备份

  ```
   $ mv /usr/local/var/postgres /usr/local/var/postgres9351
  ```

  - 初始化新数据库

  ```
    $ initdb /usr/local/var/postgres -E utf8
  ```

  - 升级

  ```
   $ pg_upgrade -b oldbindir -B newbindir -d olddatadir -D newdatadir
  ```



2. 报错信息

```
Removals:
* git_data_dir has been deprecated since 8.10 and was removed in 11.0. Use git_data_dirs instead.
```

- `git_data_dir` 写法更新，仅出现在11.0之前的老版本

  - 原语法

  ```
   git_data_dir /data/git
  ```

  - 新语法

  ```
    git_data_dirs({
       "default" => {
          "path" => "/data/git"
       }
    })
  ```

  ```
   gitlab-ctl reconfigure
  ```



## Docker 基本操作

- 查看镜像

```bash
$ docker images
```

- 打包提交

```bash
$ docker commit -a "fancy" -m "fancy‘s backup"
```

容器名称或id 打包的镜像名称:标签 

- OPTIONS说明： 
  -a :提交的镜像作者； 
  -c :使用Dockerfile指令来创建镜像； 
  -m :提交时的说明文字； 
  -p :在commit时，将容器暂停。 

例：`docker commit -p d479774b6e8e -a "fancy" -m "Gitlab before L10n" Gitlab-001`

- 备份镜像

```bash
$ docker save -o /data/images-backup/container-gitlab-backup.tar container-gitlab-backup
```

- 说明:
  `/data/images-backup/container-gitlab-backup.tar`镜像导出存放路径
  `container-gitlab-backup` 镜像在docker里面的名字

- 恢复镜像

```bash
$ docker import [OPTIONS] URL|- [REPOSITORY[:TAG]]
# 导入hello快照，并制定镜像标签为gitlab:1.0
$ cat gitlab.tar | sudo docker import - hello:1.0
```

  

## Docker Gitlab 操作

- 任意位置进入Docker内Gitlab环境

```bash
$ sudo docker exec -it gitlab /bin/bash
```



- 目录/文件

```
主配置文件：/etc/gitlab/gitlab.rb
程序安装目录：/opt/gitlab
程序配置和运行目录：/var/opt/gitlab
程序日志目录：/var/log/gitlab
```

- 命令

```
升级数据库：gitlab-rake db:migrate
显示配置环境：gitlab-rake gitlab:env:info
检查gitlab运行环境：gitlab-rake gitlab:check
gitlab安装（慎用，会清数据库）：gitlab-rake gitlab:setup
gitlab启动/停止/重启：gitlab-ctl start/stop/restart
```



## 目录与Crontab自动备份

> 这里继承我的Docker run 命令，如果改动的目录也会变更

- 备份目录

```
/gitlab/gitlabv2/data/backups
```

- 设置目录

```
/gitlab/gitlabv2/config
```

- 日志目录

```
/gitlab/gitlabv2/logs
```

- 手动备份

```BASH
$ docker exec -it gitlab bash # 进入容器
```

```bash
$ gitlab-rake gitlab:backup:create
```

- 自动备份`借助corntab，RHEL等发行版需自行安装`

  - `vi /etc/rc.d/rc.local `添加 cron 设置开机启动

    ```
    /sbin/service crond start   
    ```


  - `vi ~/gitlab_backup.sh` 编辑cron脚本

    ```
    #！ /bin/bash
    case "$1" in 
        start)
                docker exec gitlab gitlab-rake gitlab:backup:create
                ;;
    esac
    ```

  - `gitlab_backup.sh start`

  - `crontab -e`

  ```
  0 13 * * * /root/gitlab_backup.sh start
  0 18 * * * /root/gitlab_backup.sh start
  0 23 * * * /root/gitlab_backup.sh start
  ```

  - 附：cron 语法

  ```
  *  *  *  *  *  command
  分  时  日  月  周  命令
  
  其中，
  第1列表示分钟，1~59，每分钟用*表示
  第2列表示小时，1~23，（0表示0点）
  第3列表示日期，1~31
  第4列表示月份，1~12
  第5列表示星期，0~6（0表示星期天）
  第六列表示要运行的命令。
  ```

