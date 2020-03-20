---
title: "Docker搭建維護Gitlab "
date: 2020-01-20T14:10:12+08:00
draft: false

categories: ['Code']
tags: ['Crontab', 'Gitlab', 'Docker']
author: "Fancy"

resizeImages: false
---

Gitlab 從安裝到升級維護，內含 Gitlab 的維護指南，Docker基本操作，Crontab基本操作
<!--more-->



## Docker的Gitlab安裝流程

> 碎碎念：目前家裏運行的14個docker容器11個是在Debian，hassio，qbittorent，pihole，openwrt，supervisor，Don't starve Togeter Server，Minecarft Server，Netease Music API其他亂七八糟各種服務 自從有了Docker再也找不到不用的理由。

- 添加 CA 證書（Debian系）:

```bash
$ sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

- 添加 GPG 密鑰（Debian系）:

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

- 添加 source.list: Debian上遊額外安裝

```bash
$ sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
```

- 安裝 Docker CE: 

```bash
$ sudo apt-get update

$ sudo apt-get install docker-ce
```

- 啟動 Docker CE:

```bash
$ sudo service docker start
```

- 測試 Docker CE:

```bash
$ sudo service docker start
```

- Docker 構建命令 (我這裏命名為v2 )

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

# 輸入簡介

Please enter the gitlab-ci tags for this runner (comma separated):

# 幫這個runner加上tag，這tag會影響到runner要抓哪個工作來執行 (有多個以逗號分開，例如tag1,tag2)

Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:

# docker

Please enter the Docker image (eg. alpine:latest):

# 預設使用的docker-image
```





# Gitlab 跨版本遷移及更新流程

- 查詢版本

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION 
# 以最初為8.8.5為例，這差不多是我找到最早的還有大量用戶的版本了，其他的版本均適用
```

- 備份

```bash
$ sudo gitlab-rake gitlab:backup:create STRATEGY=copy  
# 備份後的文件壹般是位於/var/opt/gitlab/backups下, 自動生成文件名文件名如1541739348_2020_01_19_10.3.3_gitlab_backup.tar
```

- 關閉部分gitlab服務

```shell
$ gitlab-ctl stop unicorn
$ gitlab-ctl stop sidekiq
$ gitlab-ctl stop nginx

```

- 利用大版本號的最新版本，按照以下推薦升級順序

`8.8.5 > 8.17.7 > 9.5.10 > 10.8.7 > 11.11.8 > 12.0.12 > 12.3.5`

```bash
$ yum list gitlab-ce-10*
```

```bash
$ yum update gitlab-ce-10.8.7
# $ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-10.6.2-ce.0.el7.x86_64.rpm
```

- 同理多次，直到最後壹次

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

- 中文社區版漢化

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

- 數據遷移

```bash
$ scp -pr 1541677863_2020_01_18_11.4.5_gitlab_backup.tar root@10.101.45.12:/var/opt/gitlab/backups
# 把對應版本的數據從舊服務器上拷貝到新服務器的gitlab備份目錄裏
# 新服務器執行恢復命令
$ chown -R git.git /var/opt/gitlab/backups/
$ gitlab-rake gitlab:backup:restore RAILS_ENV=production BACKUP=1541677863_2020_01_18_11.4.5

註意：這裏沒有後面的_gitlab_backup.tar名字
壹路yes，恢復是會先刪除新服務器上所有gitlab數據的。

```



# Q&A


1. 報錯信息

```
The data directory was initialized by PostgreSQL version 9.2.18, which is not compatible with this version 9.6.2.
```

- Postgresql版本過低

  - 備份

  ```
   $ mv /usr/local/var/postgres /usr/local/var/postgres9351
  ```

  - 初始化新數據庫

  ```
    $ initdb /usr/local/var/postgres -E utf8
  ```

  - 升級

  ```
   $ pg_upgrade -b oldbindir -B newbindir -d olddatadir -D newdatadir
  ```



2. 報錯信息

```
Removals:
* git_data_dir has been deprecated since 8.10 and was removed in 11.0. Use git_data_dirs instead.
```

- `git_data_dir` 寫法更新，僅出現在11.0之前的老版本

  - 原語法

  ```
   git_data_dir /data/git
  ```

  - 新語法

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

- 查看鏡像

```bash
$ docker images
```

- 打包提交

```bash
$ docker commit -a "fancy" -m "fancy‘s backup"
```

容器名稱或id 打包的鏡像名稱:標簽 

- OPTIONS說明： 
  -a :提交的鏡像作者； 
  -c :使用Dockerfile指令來創建鏡像； 
  -m :提交時的說明文字； 
  -p :在commit時，將容器暫停。 

例：`docker commit -p d479774b6e8e -a "fancy" -m "Gitlab before L10n" Gitlab-001`

- 備份鏡像

```bash
$ docker save -o /data/images-backup/container-gitlab-backup.tar container-gitlab-backup
```

- 說明:
  `/data/images-backup/container-gitlab-backup.tar`鏡像導出存放路徑
  `container-gitlab-backup` 鏡像在docker裏面的名字

- 恢復鏡像

```bash
$ docker import [OPTIONS] URL|- [REPOSITORY[:TAG]]
# 導入hello快照，並制定鏡像標簽為gitlab:1.0
$ cat gitlab.tar | sudo docker import - hello:1.0
```

  

## Docker Gitlab 操作

- 任意位置進入Docker內Gitlab環境

```bash
$ sudo docker exec -it gitlab /bin/bash
```



- 目錄/文件

```
主配置文件：/etc/gitlab/gitlab.rb
程序安裝目錄：/opt/gitlab
程序配置和運行目錄：/var/opt/gitlab
程序日誌目錄：/var/log/gitlab
```

- 命令

```
升級數據庫：gitlab-rake db:migrate
顯示配置環境：gitlab-rake gitlab:env:info
檢查gitlab運行環境：gitlab-rake gitlab:check
gitlab安裝（慎用，會清數據庫）：gitlab-rake gitlab:setup
gitlab啟動/停止/重啟：gitlab-ctl start/stop/restart
```



## 目錄與Crontab自動備份

> 這裏繼承我的Docker run 命令，如果改動的目錄也會變更

- 備份目錄

```
/gitlab/gitlabv2/data/backups
```

- 設置目錄

```
/gitlab/gitlabv2/config
```

- 日誌目錄

```
/gitlab/gitlabv2/logs
```

- 手動備份

```BASH
$ docker exec -it gitlab bash # 進入容器
```

```bash
$ gitlab-rake gitlab:backup:create
```

- 自動備份`借助corntab，RHEL等發行版需自行安裝`

  - `vi /etc/rc.d/rc.local `添加 cron 設置開機啟動

    ```
    /sbin/service crond start   
    ```


  - `vi ~/gitlab_backup.sh` 編輯cron腳本

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

  - 附：cron 語法

  ```
  *  *  *  *  *  command
  分  時  日  月  周  命令
  
  其中，
  第1列表示分鐘，1~59，每分鐘用*表示
  第2列表示小時，1~23，（0表示0點）
  第3列表示日期，1~31
  第4列表示月份，1~12
  第5列表示星期，0~6（0表示星期天）
  第六列表示要運行的命令。
  ```

