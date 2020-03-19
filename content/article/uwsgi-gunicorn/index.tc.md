---
title: "uWSGI和Gunicorn對比實踐筆記"
date: 2019-01-27T12:13:48+08:00

tags: ['Python', 'Gunicorn', 'uWSGI']
author: "Fancy"
noSummary: true

resizeImages: false
---
在服務器能正常通信後，Werkzeug提供的的WSGI並不適合生產環境，調試完代碼第一件事就找個獨立WSGI容器。我們這裏選擇uWSGI和gunicorn，相對來說這兩個比gevent更適合大型項目，uWSGI出現的比較早，gunicorn是從Ruby的unicorn移植而來，配置也簡單。為了熟悉uwsgi我都嘗試一下吧
<!--more-->

> 從flask微信公眾號後臺分析
> Ubuntu+Python3+pipenv+flask環境下 uWSGI/Gunicorn+supervisor+nginx配置問題
>
> 這裏的示例程序代碼接上篇[使用Flask創建微信公眾號](https://link.zhihu.com/?target=https%3A//fancysly.com/post/9)

## uWSGI配置

這裏我們先配置uwsgi，或者使用gunicorn，這兩個任選其一。 如果是新浪SAE的話記得刪除app.run()裏的port，並且以下所有端口都是5050。

```bash
$ pipenv install uwsgi
```

然後運行

```bash
$ uwsgi --socket 0.0.0.0:5000 -w main:app
```

可以加上--protocol=http以指定協議，我們的項目啟動文件名為`main.py`對應修改即可，運行`http://server_domain_or_IP:5000`倘若能看到我們在視圖函數寫的`Hello World!`即可。 如果開了防火墻莫忘了開啟端口，以Ubuntu的ufw為例：

```bash
$ sudo ufw allow 5000
```

到此，可以到項目根目錄創建配置文件wepub_uwsgi.ini，這裏名字隨意。

*wepub_uwsgi.ini:*

```yaml
[uwsgi]
module = main:app
#用以啟動程序名
master = true
processes = 4
#子進程數量
chdir = /root/WePub/wepub
#python啟動程序目錄
socket = /root/WePub/wepub/wepub_uwsgi.sock
#uwsgi 啟動後所需要創建的文件，和 Nginx 通信，配置 Nginx 時用
logto = /root/WePub/wepub/%n.log
chmod-socket = 660
#賦予 .sock 文件權限與 Nginx 通信
vacuum = true
http = 0.0.0.0:5050
#http地址和端口
```

保存後運行：

```bash
$ uwsgi --ini wepub_uwsgi.ini
```

此時**運行時**會在設置的目錄生成與配置文件同名的.sock文件，先不要關閉，等配置好Nginx後方可關閉確認後跳過gunicorn章節，即配置Nginx。

## gunicorn配置

個人偏好於gunicorn，用的比較多了，至於區別，gunicorn比較好部署理解，性能上的短板還在我身上，安裝gunicorn

```bash
$ pipenv install gunicorn
```

運行

```bash
$ gunicorn -w 4 -b 0.0.0.0:8000 main:app
```

其中-w 是--workers是定義同步worker的數量，--bind為主機地址和端口，不用異步的話這就夠了 gunicorn的配置文件還能實現復雜的配置，目前學有不精，先不做討論

## uWSGI配置nginx

nginx反向代理使用比較多了，流程都比較一致，這裏就簡潔帶過吧

```bash
$ sudo apt install nginx
```

這裏我們先把nginx的配置文件找個位置備份好

```bash
$ mv /etc/nginx/sites-enabled/default ~/nginx.backup
```

而後直接定義server塊，listen監聽80端口 (新浪SAE為5050端口）

```bash
$ vi /etc/nginx/sites-enabled/wepub
```

*wepub:*

```json
server {
    listen  80;
    server_name _;

    location / {
    include      uwsgi_params;
    uwsgi_pass unix:/root/WePub/wepub/wepub_uwsgi.sock;
    }
}
```

`server_name`可以指定HOST域名解析後的公網地址，這裏默認用的是通配符 這裏是最簡配置，定義日誌存放重寫請求頭部這些，有機會再詳細說明吧 如果項目有靜態文件的話，設置路徑與過期時間

```json
location /static {
    alias /the/path/to/staitic;
    expires 30d;
}
```

保存退出，最後 sudo nginx -t 檢查語法錯誤，期待返回

```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

有錯誤的話檢查一下，而後重啟nginx使配置生效

```bash
$ sudo service nginx restart
```

## gunicorn配置nginx

只是配置文件不同

*wepub:*

```json
server {
    listen 80;
    server_name example.org;


    location / {
    proxy_pass http://127.0.0.1:8000; #轉發gunicorn運行地址
    }
 }
```

location 只需要直接轉發gunicorn運行地址即可運行，這裏是最簡配置，成功後直接80地址即可訪問服務器反向代理部分：

```json
    location / {
        proxy_pass         http://localhost:8000/; #gunicorn 運行地址
        proxy_redirect     off;

        proxy_set_header   Host             $http_host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
```



## 管理uWSGI進程：

這裏的差別主要就各自的命令而已，就放在一起說吧 對於uWSGI來說，uWSGI服務與nginx通信時依賴於sock文件，網站運行時無法關閉，始終不可靠，這裏我們利用Supervisor管理並監控uWSGI的進程，

```bash
$ sudo apt install supervisor
```

這裏我們不更改全局配置，為我們的程序單獨創建配置文件

```bash
$ sudo vi /etc/supervisor/conf.d/wepub.conf
```

*wepub.conf:*

```yaml
[program:wepub]
command = pipenv run uwsgi /root/WePub/wepub/wepub_uwsgi.ini ;監控uWSGI
# command=pipenv run gunicorn -w 4 main:app       監控Gunicorn
directory=/root/WePub/wepub                    ;程序目錄
user=root                    ; 程序啟動用戶，最好不要用root
autostart=true                 ; 隨supervisor啟動而啟動
autorestart=true              ; 自動重啟
stopasgroup=true           ; 關閉程序時關閉子進程
killasgroup=true             ; 殺死程序時殺死子進程
stderr_logfile=/root/WePub/wepub/err.log   ;錯誤日誌文件
stdout_logfile=/root/WePub/wepub/out.log  ;程序日誌文件
stopsignal=INT          ; 殺死進程的自定信號
startretries=10               ; 啟動失敗時的重試次數

[supervisord]
```

我們:wq保存後檢查配置文件是否正確：

```bash
$ supervisord -c /path/to/my/config.conf
```

返回的錯誤信息挺詳細的，根據問題修改即可。確認無返回的時候重啟supervisor

```bash
$ sudo service supervisor restart
```

如果運行過程中出現錯誤，可以查看supervisor的錯誤日誌。也可以在`supervisorctl`命令行中進行查看與操作程序，也可以開啟9001瀏覽器訪問。

------

瀏覽器輸入Nginx顯示502badway，一般這時候是防火墻沒打開或者sock文件位置有誤

提示`unix:///var/run/supervisor.sock no such file`除了supervisor安裝問題之外， 一般來說是指定位置或者和權限的問題

```yaml
[unix_http_server]
file=/var/run/supervisor.sock   ; (socket 文件位置)
chmod=0700                       ; sockef file mode (default 0700)
```

如果是提示編碼出錯還需解決pipenv的編碼問題

```bash
$ sudo vi /etc/supervisor/supervisord.conf
```

在[supervisord]的最後添加環境設置：

```bash
environment=LC_ALL='zh_CN.UTF-8', LANG='zh_CN.UTF-8'
```

- 不知道新浪SAE怎麽搞得非要debian系限定的C.UTF-8，我想這並不是個好辦法

![img](https://pic4.zhimg.com/80/v2-d553a36c276824c97c686d079d873d33_1440w.jpg)

```bash
environment=LC_ALL='C.UTF-8', LANG='C.UTF-8'
```

至此應該能得到一個還算穩定安全的部署程序了，應該足以應對個人微信公眾號這些並發不高無需協程的了。 關於每個部分的詳細設置都有很多，還是詳細閱讀文檔吧。這裏如果遇到什麽問題的話還請指教