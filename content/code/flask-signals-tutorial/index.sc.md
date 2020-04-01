---
title: "使用Flask的Signals"
date: 2019-12-26T15:34:53+08:00
draft: false

categories: ['Code']
tags: ['Signal', 'Blinker', 'Python']
author: "Fancy"
---
Blinker触发信号记录与实践

<!--more-->



blinker 是 python 语言中一个强大的信号库， 提供了一些非常有用的特性，支持命名空间、匿名信号,弱引用实现与接收者之间的自动断开连接以及指定发送者,接收返回值等，同时具有线程安全的特性. (回顾之后, 发现这些特性对于我们的实现至关重要)

早期的Flask（0.6+）版本开始,Flask就集成了一个基于blinker的Signal库。并没有太多人使用, 但却是是一个很有以哦那个的功能. 类似于类Unix系统里的Signal, 同样是基于发布-订阅（Publish/Subscribe）的观察者（Observer）模式的设计模式. 

在没有安装blinker库时Flask内置的Singal 可以提供支持，但注意目前的版本如果不安装blinker的话Signal是无法使用的，blinker可通过pip包进行安装`pip3 install blinker`

通过Ipython测试是否正确的调用了blinker库

```python
In [1]: from flask import signals

In [2]: signals.signals_available
Out[2]: True   # 如果信号系统不可用（未安装Blinker）则为False
```

至此，我们就可以在项目中使用Signal系统了。



## Flask内置信号

在项目中使用时，Flask已经内置了一些Signal，下表展示了Flask内置的Signals，详细请参考[Flask built-in signals](https://link.jianshu.com?t=http://flask.pocoo.org/docs/0.12/api/#core-signals-list):

| Signals                 | 说明                                                        |
| ----------------------- | ----------------------------------------------------------- |
| template_rendered       | 模版成功渲染之后触发                                        |
| before_render_template  | 模版渲染之前触发                                            |
| request_started         | 请求上下文建立之后，请求被处理之前触发                      |
| request_finished        | 响应发送给客户端之前被触发                                  |
| got_request_exception   | 处理过程中发生异常时（早于程序异常处理，Debug模式也会）触发 |
| request_tearing_down    | 请求中断时（即使发生异常也会）触发                          |
| appcontext_tearing_down | 应用上下文中断时触发                                        |
| appcontext_pushed       | 推送应用上下文时触发，由应用发送（单元测试常用）            |
| appcontext_popped       | pop弹出应用上下文时触发，由应用发送                         |
| message_flashed         | 应用发送消息时触发                                          |

这些内置的信号模块，可以直接在回调函数通过 from flask import 导入，通过`connect()`方法进行订阅，用`disconnect()`方法来退订信号。要确保订阅一个信号，请确保也提供一个发送者以防监听全部应用的信号



### 订阅

比如一个找出模板被渲染和传入模板的变量的助手的订阅端:

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

客户端:

```
with captured_templates(app) as templates:
    rv = app.test_client().get('/')
    assert rv.status_code == 200
    assert len(templates) == 1
    template, context = templates[0]
    assert template.name == 'index.html'
    assert len(context['items']) == 10
```

通过注册一个回调函数进行接受，在测试完成后，再取消注册该回调函数，这种方法在单元测试中非常好用。

在定义回调函数时，第一个参数必须是信号发出要调用的函数（信号发送对象），第二个参数**extra是可选的， 用于接受额外的参数。

Blinker 1.4 新增了方便的方法`connected_to()`，允许使用上下文管理器把函数临时订阅到信号。由于上下文管理器的返回值是指定的，故必须将列表作为参数传递。

订阅的用法:

```
from flask import template_rendered

def captured_templates(app, recorded, **extra):
    def record(sender, template, context):
        recorded.append((template, context))
    return template_rendered.connected_to(record, app)
```

之前那个助手的例子会像这样:

```
templates = []
with captured_templates(app, templates, **extra):
    ...
    template, context = templates[0]
```



我们举个完整的例子:

发送示例：

```python
def render_templete(template, context, app):
    res = template.render(context)
    template_rendered.send(app, template=template, context=context)
    return res
```

订阅示例:

```python
def log_template_renders(sender, template, context, **extra):
    sender.logger.debug('Rendering template "%s" with context %s',
                        template.name or 'string template',
                        context)

from flask import template_rendered
template_rendered.connect(log_template_renders, app)
```

如果以上的Signal无法满足需求，Flask还提供了命名空间用于自定义信号， 限制就是只支持send方式进行传递信号，如果调用其他的操作（包括connecting）将会抛出RuntimeError异常。

# Blinker 自定义信号



我们这里可以直接使用Blinker库，在大型Flask项目结构中，通常当作插件调用，我们可以在存放插件实例化的地方定义命名空间，例：`extensions.py`

```python
from blinker import Namespace

my_signals = Namespace()

```

此时，我们就可以使用自定义的命名空间定义signal了， 根据需求也可以定义多个，这里个人假设以`Blueprint`蓝本为单位定义多个信号, 以订阅特定发布者。

```python
model_saved = my_signals.signal('model-saved')
```

```python
...
auth_log = log_signals.signal("auth_log") # 用户验证蓝本
v1_log = log_signals.signal("v1_log")     # RESTful API 蓝本
main_log = log_signals.signal("main_log") # 主路由蓝本
```

如果是编写插件的话，调用`flask.signals.Namespace`类可以脱离Blinker的依赖限制。

### 发送

可以通过`send()`方法进行传递信号, 

```python
class Model(objext):
    ...

    def model_test(self):
        main_log.send(self)
```



如同`Blinker`提供的那样， `send()`同样自由的支持额外传递关键字参数给订阅者。

如果有信号的类，把 `self` 作为发送者。如果你从一个随机的函数发出信号，发送者则为`current_app._get_current_object()`。

例:

```python
from flask import current_app

def liker_article(article, user_id):
    ...
    liker_article.send(current_app._get_current_object())
```



我们以蓝本中登录登出的路由为例, 传递区分的自定义信号来告知订阅者用户的访问路由行为。

例:`blueprint/auth.py`

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
API的验证逻辑
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
    ... # 验证逻辑
        ref = request.base_url
        ip = request.remote_addr #获取登录IP
        auth_log.send(inspect.stack()[0][3], track="use test login_route login success", user=g.user, ip=ip)  # 发送登录信号
    return redirect(url_for('main.home'))


@auth_bp.route('/logout', methods=['GET'])
@auth.login_required
def logout():
"""
logout method
"""
    session.pop(g.user, None)
    ip = request.remote_addr
    auth_log.send(inspect.stack()[0][3], track="logout success", user=g.user, ip=ip) # 发送登出信号
    g.user = None
    ret = "logout success"
    return jsonify(ret)

...

```

### 订阅

默认情况下任何发布者的信号都会传送给订阅者,不会加以区分, 这里我们为了区分我们可以给`Signal.connect()`传递一个参数，实现订阅者与订阅特定的发送者。

```python
def auth_subscriber(sender):
    print("Got an auth signal")
    assert sender.name == "auth"
   
>>> auth_log.connect(auth_subscriber, sender="anomalous")
```



更简单的, 我更加建议使用装饰器的形式，同样能达到区分发送者的效果, 样例中我们简单定义一个日志，记录每个用户的登录IP使用插件实例化的db（Flask-SQLAlchemy）使用ORM语句写入数据库

这个例子中用了一个装饰器`@connect`，除了 `@connect` ,Blinker 1.1还新增了 `@connect_via(app)`来简化此过程，AOP设计模式不得不说方便了许多, 轻松的订阅指定的信号

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



> 在简单的Flask内部的消息分发中,例如日志记录,  路由通知, Signal的实用性显现出来, 倘若是更简单的改变行为类要求, 就使用请求钩子(Hooks)吧,如果涉及Flask外调用或者更加复杂的消息通信时,不妨考虑使用Redis或者RabbitMQ, Kafka等消息队列, 消息中间件实现.