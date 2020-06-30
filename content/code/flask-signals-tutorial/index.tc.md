---
title: "使用Flask的Signals"
date: 2019-12-26T15:34:53+08:00
draft: false

categories: ['Code']
tags: ['Signal', 'Blinker', 'Python']
author: "Fancy"
resizeImages: false
---
Blinker觸發信號記錄與實踐

<!--more-->



blinker 是 python 語言中壹個強大的信號庫， 提供了壹些非常有用的特性，支持命名空間、匿名信號,弱引用實現與接收者之間的自動斷開連接以及指定發送者,接收返回值等，同時具有線程安全的特性. (回顧之後, 發現這些特性對於我們的實現至關重要)

早期的Flask（0.6+）版本開始,Flask就集成了壹個基於blinker的Signal庫。並沒有太多人使用, 但卻是是壹個很有以哦那個的功能. 類似於類Unix系統裏的Signal, 同樣是基於發布-訂閱（Publish/Subscribe）的觀察者（Observer）模式的設計模式. 

在沒有安裝blinker庫時Flask內置的Singal 可以提供支持，但註意目前的版本如果不安裝blinker的話Signal是無法使用的，blinker可通過pip包進行安裝`pip3 install blinker`

通過Ipython測試是否正確的調用了blinker庫

```python
In [1]: from flask import signals

In [2]: signals.signals_available
Out[2]: True   # 如果信號系統不可用（未安裝Blinker）則為False
```

至此，我們就可以在項目中使用Signal系統了。



## Flask內置信號

在項目中使用時，Flask已經內置了壹些Signal，下表展示了Flask內置的Signals，詳細請參考[Flask built-in signals](https://link.jianshu.com?t=http://flask.pocoo.org/docs/0.12/api/#core-signals-list):

| Signals                 | 說明                                                        |
| ----------------------- | ----------------------------------------------------------- |
| template_rendered       | 模版成功渲染之後觸發                                        |
| before_render_template  | 模版渲染之前觸發                                            |
| request_started         | 請求上下文建立之後，請求被處理之前觸發                      |
| request_finished        | 響應發送給客戶端之前被觸發                                  |
| got_request_exception   | 處理過程中發生異常時（早於程序異常處理，Debug模式也會）觸發 |
| request_tearing_down    | 請求中斷時（即使發生異常也會）觸發                          |
| appcontext_tearing_down | 應用上下文中斷時觸發                                        |
| appcontext_pushed       | 推送應用上下文時觸發，由應用發送（單元測試常用）            |
| appcontext_popped       | pop彈出應用上下文時觸發，由應用發送                         |
| message_flashed         | 應用發送消息時觸發                                          |

這些內置的信號模塊，可以直接在回調函數通過 from flask import 導入，通過`connect()`方法進行訂閱，用`disconnect()`方法來退訂信號。要確保訂閱壹個信號，請確保也提供壹個發送者以防監聽全部應用的信號



### 訂閱

比如壹個找出模板被渲染和傳入模板的變量的助手的訂閱端:

```
from flask import template_rendered
from contextlib import contextmanager

@contextmanager
def captured_templates(app):
    recorded = []
    def record(sender, template, context, **extra):
        recorded.append((template, context))
    template_rendered.connect(record, app)
    try:
        yield recorded
    finally:
        template_rendered.disconnect(record, app)
```

客戶端:

```
with captured_templates(app) as templates:
    rv = app.test_client().get('/')
    assert rv.status_code == 200
    assert len(templates) == 1
    template, context = templates[0]
    assert template.name == 'index.html'
    assert len(context['items']) == 10
```

通過註冊壹個回調函數進行接受，在測試完成後，再取消註冊該回調函數，這種方法在單元測試中非常好用。

在定義回調函數時，第壹個參數必須是信號發出要調用的函數（信號發送對象），第二個參數**extra是可選的， 用於接受額外的參數。

Blinker 1.4 新增了方便的方法`connected_to()`，允許使用上下文管理器把函數臨時訂閱到信號。由於上下文管理器的返回值是指定的，故必須將列表作為參數傳遞。

訂閱的用法:

```
from flask import template_rendered

def captured_templates(app, recorded, **extra):
    def record(sender, template, context):
        recorded.append((template, context))
    return template_rendered.connected_to(record, app)
```

之前那個助手的例子會像這樣:

```
templates = []
with captured_templates(app, templates, **extra):
    ...
    template, context = templates[0]
```



我們舉個完整的例子:

發送示例：

```python
def render_templete(template, context, app):
    res = template.render(context)
    template_rendered.send(app, template=template, context=context)
    return res
```

訂閱示例:

```python
def log_template_renders(sender, template, context, **extra):
    sender.logger.debug('Rendering template "%s" with context %s',
                        template.name or 'string template',
                        context)

from flask import template_rendered
template_rendered.connect(log_template_renders, app)
```

如果以上的Signal無法滿足需求，Flask還提供了命名空間用於自定義信號， 限制就是只支持send方式進行傳遞信號，如果調用其他的操作（包括connecting）將會拋出RuntimeError異常。

# Blinker 自定義信號



我們這裏可以直接使用Blinker庫，在大型Flask項目結構中，通常當作插件調用，我們可以在存放插件實例化的地方定義命名空間，例：`extensions.py`

```python
from blinker import Namespace

my_signals = Namespace()

```

此時，我們就可以使用自定義的命名空間定義signal了， 根據需求也可以定義多個，這裏個人假設以`Blueprint`藍本為單位定義多個信號, 以訂閱特定發布者。

```python
model_saved = my_signals.signal('model-saved')
```

```python
...
auth_log = log_signals.signal("auth_log") # 用戶驗證藍本
v1_log = log_signals.signal("v1_log")     # RESTful API 藍本
main_log = log_signals.signal("main_log") # 主路由藍本
```

如果是編寫插件的話，調用`flask.signals.Namespace`類可以脫離Blinker的依賴限制。

### 發送

可以通過`send()`方法進行傳遞信號, 

```python
class Model(objext):
    ...

    def model_test(self):
        main_log.send(self)
```



如同`Blinker`提供的那樣， `send()`同樣自由的支持額外傳遞關鍵字參數給訂閱者。

如果有信號的類，把 `self` 作為發送者。如果妳從壹個隨機的函數發出信號，發送者則為`current_app._get_current_object()`。

例:

```python
from flask import current_app

def liker_article(article, user_id):
    ...
    liker_article.send(current_app._get_current_object())
```



我們以藍本中登錄登出的路由為例, 傳遞區分的自定義信號來告知訂閱者用戶的訪問路由行為。

例:`blueprint/auth.py`

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
API的驗證邏輯
"""


import inspect
from backend.extensions import auth
from flask import Blueprint,  url_for, redirect, request, jsonify, g, session
from backend.util import auth_log

auth_bp = Blueprint('auth', __name__)


@auth_bp.route('/login', methods=['POST', 'GET'])
def login():
"""
login method
"""    
    if request.method == "POST":
    ... # 驗證邏輯
        ref = request.base_url
        ip = request.remote_addr #獲取登錄IP
        auth_log.send(inspect.stack()[0][3], track="use test login_route login success", user=g.user, ip=ip)  # 發送登錄信號
    return redirect(url_for('main.home'))


@auth_bp.route('/logout', methods=['GET'])
@auth.login_required
def logout():
"""
logout method
"""
    session.pop(g.user, None)
    ip = request.remote_addr
    auth_log.send(inspect.stack()[0][3], track="logout success", user=g.user, ip=ip) # 發送登出信號
    g.user = None
    ret = "logout success"
    return jsonify(ret)

...

```

### 訂閱

默認情況下任何發布者的信號都會傳送給訂閱者,不會加以區分, 這裏我們為了區分我們可以給`Signal.connect()`傳遞壹個參數，實現訂閱者與訂閱特定的發送者。

```python
def auth_subscriber(sender):
    print("Got an auth signal")
    assert sender.name == "auth"
   
>>> auth_log.connect(auth_subscriber, sender="anomalous")
```



更簡單的, 我更加建議使用裝飾器的形式，同樣能達到區分發送者的效果, 樣例中我們簡單定義壹個日誌，記錄每個用戶的登錄IP使用插件實例化的db（Flask-SQLAlchemy）使用ORM語句寫入數據庫

這個例子中用了壹個裝飾器`@connect`，除了 `@connect` ,Blinker 1.1還新增了 `@connect_via(app)`來簡化此過程，AOP設計模式不得不說方便了許多, 輕松的訂閱指定的信號

例：`util.py`

```python
from backend.extensions import auth_log, main_log, v1_log, db

...

# Singal
@auth_log.connect
def auth_subscriber(sender,**kwargs):
    user = kwargs.get("user")
    if user:
        print(f'Got auth signal sent by {sender}')
        track = kwargs.get("track", "")
        level = sender
        ip = kwargs.get("ip", "")
        log = Log(user=str(user), login_ip=str(ip), level=str(level), track=str(track))
        db.session.add(log)
        db.session.commit()
```



> 在簡單的Flask內部的消息分發中,例如日誌記錄,  路由通知, Signal的實用性顯現出來, 倘若是更簡單的改變行為類要求, 就使用請求鉤子(Hooks)吧,如果涉及Flask外調用或者更加復雜的消息通信時,不妨考慮使用Redis或者RabbitMQ, Kafka等消息隊列, 消息中間件實現.