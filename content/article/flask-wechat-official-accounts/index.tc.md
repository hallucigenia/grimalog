---
title: "使用Flask創建微信公眾號"
date: 2018-11-26T13:07:13+08:00
categories: ['Code']
tags: ['Flask', 'Python', 'Wechat']
author: "Fancy"

resizeImages: false
---

基於Flask搭建微信公眾號後台

<!--more-->

> 這次先用Flask為微信公眾號做個後台。微信公眾號後台一般對性能各方面要求並不高，這裏我們以新浪SAE為例，其他已解析域名的服務器同理。整個過程比較簡單，算是個快速的小項目吧

- 部署環境為python3+pipenv+flask+uwsgi/gunicorn+supervisor+nginx 其中我們uwsgi和gunicorn我會同步對比部署，篇幅太長主題不明，主要內容留在下篇，這裏先說實現的問題吧

## **部署服務器**

------

可以點擊鏈接進行註冊 說起新浪SAE(SinaAppEngine)類屬Paas，這個還是比較不穩定的，好在前期不花錢，自帶二級域名和HTTPS，控制台直接操作ssh秘鑰，不支持系統內ssh密鑰相比之下很適合做測試和微信這些。

[註冊地址](https://link.zhihu.com/?target=http%3A//www.sinacloud.com/public/login/inviter/gaimrn-mddmzeKWrhKWnaYGuibB-no1qfnyudg.html) 新浪SAE有提供免費的基於GIT的Python2.7共享環境，這裏不用這個，實在是太難受了，我們選擇自定義一個，選擇手工部署Ubuntu，倘若Centos的弄nginx這些有點煩。



![img](https://pic2.zhimg.com/80/v2-e555aa9c29772b5b8f038072259d7309_1440w.jpg)

新浪雲的Ubuntu新建用戶後是無法ssh秘鑰登陸的，在控制台改ssh秘鑰。就忍受著用root用戶吧。其他服務器用戶還是新建用戶比較安全。寫文章雖然用的新浪雲示範，往後的代碼裏為了區別權限都加上`sudo`

如果是新浪SAE記下自定義的二級域名，等下會用到

![img](https://pic3.zhimg.com/80/v2-c82a680bde155a4eb6236f5c948020d6_1440w.jpg)



管理可以點控制台/應用/容器管理。這裏我們註意框住的這句話，無論80還是443，新浪只開放了`5050`端口給我們。HTTPS也通用。這裏後面都會用到

![img](https://pic1.zhimg.com/80/v2-f99e3484018af4155a4d24f56fc87b88_1440w.jpg)



## **接入微信公眾號**

------

進入微信公眾平台，如果沒有公眾號的話按[流程申請](https://link.zhihu.com/?target=https%3A//kf.qq.com/faq/120911VrYVrA151009eIrYvy.html)即可，從主頁中下滑，在左導航欄中最下面進入到 / 開發 / 基本配置界面。勾選協議成為開發者，





![img](https://pic3.zhimg.com/80/v2-51654ad92fbe19b96d525aa90d334352_1440w.jpg)



點擊“修改配置”按鈕，填寫服務器地址（URL）、Token和EncodingAESKey，URL是與服務器通信接口的接口URL，我們填入服務器的解析域名（SAE的二級域名）後跟一個自定義路由，http和https都可以，新浪雲自帶有https這是沒問題的。Token可由開發者自由填寫，用作生成簽名（該Token會和接口URL中包含的Token進行哈希值比對驗證安全性）將用作消息體加解密密鑰。我們點隨機生成，

微信後台接受一個GET請求需要的參數如下：

![img](https://pic4.zhimg.com/80/v2-8c05612b193ba1a7c6e7651fdf17dd5f_1440w.jpg)

> 1）將token、timestamp、nonce三個參數進行字典序排序 2）將三個參數字符串拼接成一個字符串進行sha1加密 3）開發者獲得加密後的字符串可與signature對比，標識該請求來源於微信

據微信的要求，我們嘗試編寫flask後台。先部署下基本環境

```bash
$ sudo apt update & upgrade
$ sudo apt install python3-dev python3-pip
$ sudo pip3 install pipenv
```

不是新浪SAE強烈建議創建`adduser`新用戶，並`usermod -aG sudo`賦予權限。創建ssh密鑰登錄並關閉密碼登錄。 創建pipenv環境並進入安裝flask這些就不多說了。如果未測試的話可以先簡單編寫個測試

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

 @app.route('/wechat_api/')
 def wechat():
    pass

if __name__ == "__main__":
    app.run(debug=True)
    # app.run(host='0.0.0.0', port=5050)
```

新浪SAE需要指定5050端口，Flask運行app.run()是默認的127.0.0.1:5000, 我們使用註釋裏的指定host參數即可外網訪問。這是我們輸入域名進入，可以看到`Hello World!`即可。

創建一個接口，參考微信提供的php編寫flask，填入token和自定義的路由，

*base.py：*

```python
import hashlib

from flask import Flask, request, make_response
import xml.etree.ElementTree as ET

WX_TOKEN = 'fancy'
# 這裏填寫公眾號配置的token

app = Flask(__name__)
app.debug = True


@app.route("/")
def hello():
    return "Hello World!"

@app.route('/wechat_api/', methods=['GET', 'POST'])
# 定義路由地址請與URL後的保持一致
def wechat():
    if request.method == 'GET':
    token = WX_TOKEN
    data = request.args
    signature = data.get('signature', '')
    timestamp = data.get('timestamp', '')
    nonce = data.get('nonce', '')
    echostr = data.get('echostr', '')
    s = sorted([timestamp, nonce, token])
    # 字典排序
    s = ''.join(s)
    if hashlib.sha1(s.encode('utf-8')).hexdigest() == signature:
        # 判斷請求來源，並對接受的請求轉換為utf-8後進行sha1加密
        response = make_response(echostr)
        # response.headers['content-type'] = 'text' 
        # 新浪SAE未實名用戶加上上面這句
        return response

if __name__ == '__main__':
app.run()
# app.run(host='0.0.0.0', port=5050)
```

然後回到微信服務器配置的地方選擇明文模式或者兼容模式。點選提交，



![img](https://pic1.zhimg.com/80/v2-57e77aa10d663885f708d588cb7f89e0_1440w.jpg)



我遇到問題。由於我新浪SAE未實名還未審核通過，新郎SAE對未實名的在返回值內會帶上奇怪的的html內容信息，從而導致Token驗證失敗。我們在返回值的頭部帶上`'content-type' = 'text'`即可。

到此，點擊提交，應該就沒問題了，如果顯示Token驗證失敗，請回頭再次檢查一遍

## **接入後的應答**



我們查看微信的文檔，微信是這樣說的：

[微信收到的消息類型結構mp.weixin.qq.com](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/wiki%3Ft%3Dresource/res_main%26id%3Dmp1421140453)[微信被動回覆用戶消息結構mp.weixin.qq.com](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/wiki%3Ft%3Dresource/res_main%26id%3Dmp1421140543)

還有一個接收事件推送，介於篇幅問題，這個留作下下次再說。 微信提供給開發者的通信接口是xml格式的，確實一直處理json，用xml還是感覺比較難受。這裏使用 xml.etree.ElementTree 來解析。

分類有文本`text`，圖片`image`，語音`voice`，視頻`video`，小視頻`shortvideo`，地理位置`location`，鏈接`link`。 這裏由於沒有太多硬性的功能，我希望簡單化用戶的操作，將功能全部先融合在輸入框，根據用戶輸入的內容做出相應的判斷，如果無法判斷時或者沒實際意義時，判斷你可能只想聊天，再調用聊天機器人。

思路是初步判斷消息類型，然後再逐個if-elif篩選下來。 - 首先判斷文字模塊，微信POST過來的XML數據包結構：

![img](https://pic1.zhimg.com/80/v2-a7c40e3249a96afff0f8d392dbfdc5b8_1440w.jpg)

判斷一個`MsgType`，主要用到的`ToUserName`，`FromUserName`，`Content`，我們先看如何回覆文本消息：

![img](https://pic4.zhimg.com/80/v2-dda1520952aeccaf1878100378a89f5f_1440w.jpg)

那麽好了，`ToUserName`，`FromUserName`實際在收發過程中是調轉的，利用time()生成整型時間，我們先把結構起好 微信這裏原來的GET請求驗證不能刪除，除此之外的POST我們直接else處理即可。

## **最開始和最後——圖靈機器人**

*main.py:*

```python
from tuling import get_response
import xml.etree.ElementTree as ET
# ...
@app.route('/wechat_api/', methods=['GET', 'POST'])
def wechat():
    if request.method == 'GET':
    #...
    else:
        xml = ET.fromstring(request.data)
        toUser = xml.find('ToUserName').text
        fromUser = xml.find('FromUserName').text
        msgType = xml.find("MsgType").text

        if msgType == 'text':
            content = xml.find('Content').text
            return reply_text(
                fromUser, toUser, get_response(
                    fromUser, content))
        else:
            return reply_text(fromUser, toUser, "嗯？我聽不太懂")
```

微信要求必須回覆success（建議）或者空字符串防止輪詢，我這裏先將未分類的消息回覆text這樣感覺不會不理人。get_response 先判斷一個`text`消息，接收`Content`再做判斷是否為關鍵語句再針對回覆。倘若沒有即調用`圖靈機器人`。 我們用的v2的api，用的是post請求一個json，json比較簡單，這邊不多說了，按[API V2.0接入文檔](https://link.zhihu.com/?target=https%3A//www.kancloud.cn/turing/www-tuling123-com/718227)的示例即可

*tuling.py:*

```python
import os
import json
import requests

TULING_KEY = os.getenv('TULING_KEY')


def get_response(openid, msg):
    api = 'http://openapi.tuling123.com/openapi/api/v2'
    dat = {
        "perception": {
            "inputText": {
                "text": msg
            },
            "inputImage": {
                "url": "imageUrl"
            },
            "selfInfo": {
                "location": {
                    "city": "北京",
                    "province": "北京",
                    "street": "信息路"
                }
            }
        },
        "userInfo": {
            "apiKey": TULING_KEY,
            "userId": openid
        }
    }
    dat = json.dumps(dat)
    r = requests.post(api, data=dat).json()

    mesage = r['results'][0]['values']['text']
    print(r['results'][0]['values']['text'])
    return mesage
```

輸出我們暫時只要`results`裏的`values`輸出值，取值範圍是文本`text`，後期再一點點補充。 APIkey可以在這裏找到

![img](https://pic2.zhimg.com/80/v2-1e97cd41c0b32618cadd4a5258e9121d_1440w.jpg)

\- 圖靈機器人支持直接接入公眾號，這件事就當我不知道 - 更新：圖靈機器人支持關鍵詞回覆，可以添加各種語料庫。這裏目前沒有回覆覆雜消息的需求就先簡化代碼。

回覆微信的xml格式按照要求處理即可， *main.py:*

```python
import time
from flask import  make_response
# ...
def reply_text(to_user, from_user, content):
    reply = """
    <xml><ToUserName><![CDATA[%s]]></ToUserName>
    <FromUserName><![CDATA[%s]]></FromUserName>
    <CreateTime>%s</CreateTime>
    <MsgType><![CDATA[text]]></MsgType>
    <Content><![CDATA[%s]]></Content>
    <FuncFlag>0</FuncFlag></xml>
    """
    response = make_response(reply % (to_user, from_user,
                                      str(int(time.time())), content))
    response.content_type = 'application/xml'
    return response
```

至此本地測試通過後用git方式什麽都好推送到服務器端，運行python main.py應該手機端發送就沒問題的。

![img](https://pic2.zhimg.com/80/v2-cf0b1fd33a5f472ced550e440089bb1d_1440w.jpg)

附：官方提供的

[微信公眾平台接口調試工具mp.weixin.qq.com](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/debug/cgi-bin/apiinfo%3Ft%3Dindex%26type%3D%E8%87%AA%E5%AE%9A%E4%B9%89%E8%8F%9C%E5%8D%95%26form%3D%E8%87%AA%E5%AE%9A%E4%B9%89%E8%8F%9C%E5%8D%95%E5%88%9B%E5%BB%BA%E6%8E%A5%E5%8F%A3%20/menu/creat)

這才剛剛開始，下一篇裏我針對性的嘗試探究下程序必須的措施`pipenv+uwsgi/gunicorn+supervisor+nginx`的部署，部署幾次都沒好好總結