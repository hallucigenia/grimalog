---
title: "利用Redis做Flask缓存后端"
date: 2019-04-01T17:22:21+08:00

categories: ['Code']
tags: ['Flask', 'Python', 'Redis']
author: "Fancy"
draft: false
resizeImages: false
---

Flask项目中使用Flask-Caching实现Redis缓存
<!--more-->

> NoSQL数据库中的Redis，其key-value支持的value类型相当丰富，数据库完全保存在内存里，支持使用硬盘持久化，甚至为数据结构服务器。基于Redis集群原理后期还可以做分布式缓存

# 使用 Flask-Caching

在这里我们调用[Flask-Caching](https://github.com/thadeusb/flask-cache)以更加便利的实现Redis的后端缓存。这次就以本Blog为例，典型化介绍Redis缓存的实现。
Flask-Caching基本的功能继承于flask-cache，大致可以分为

- 记忆参数型缓存：由@cached装饰器实现 
- 无记忆参数型缓存：由@memoize装饰器实现

*Flask-Caching是Flask-Cache的fork版本，原来的Flask-Cache已经四年没更新了，最后一个版本只支持到python 3.3*


    pipenv install flask-caching

如果没有安装redis必须的
    
    pipenv install redis

工厂函数里实例化Cache类，创建一个cache对象，如果没实例化redis也是必须的(这里我把拓展工厂函数分开了，根据自身情况做相应更改即可)：
*extensions.py :*

    from redis import Redis
    from flask_caching import Cache
    
    # ...
    redis = Redis()
    cache = Cache(config={
        'CACHE_TYPE': 'redis',
        'CACHE_KEY_PREFIX': 'fancycache',
        'CACHE_REDIS_URL': 'redis://localhost:6379'
        })

这里只是实例，Flask-Caching除了调用redis缓存后端之外还可以使用本地Python字典，uwsgi的缓存框架，mamcached等等，详见[官方文档](https://pythonhosted.org/Flask-Caching/#configuring-flask-caching)

其中`RedisCache`有以下字段可用：

| CACHE_TYPE': 'redis'  |  Flask-Caching调用redis缓存后端必须的字段 |
| :------------ | :------------ |
|  CACHE_DEFAULT_TIMEOUT | 未指定缓存超时时间时的默认缓存超时时间，默认为300秒  |
|   CACHE_KEY_PREFIX | 在所有鍵之前添加的前綴。让不同的應用程序可以使用相同的服务器。  |
|  CACHE_REDIS_HOST | Redis的服务主机，未指定则为`localhost`即`127.0.0.1`  |
|  CACHE_REDIS_PORT  |  Redis的服务端口，默认即Redis默认为`6379` |
|  CACHE_REDIS_PASSWORD | Redis如指定密码时的设置  |
| CACHE_REDIS_DB  | Redis的db库（基于零号索引），默认为零  |
| CACHE_ARGS  | 在缓存类实例化的时候会对该列表进行解包拆分并传递。  |
|   CACHE_OPTIONS |  在缓存类实例化的时候会传递该字典 |
|   CACHE_REDIS_URL | 连接到Redis服务器的URL，如：`redis://user:password@localhost:6379/2. `相当于把前面的几点整合一下，  |

传入参数并初始化拓展，注册到app对象：
*__init__.py :*

    from cryptic.extensions import cache
    # ...
    def register_extensions(app):
        # ...
        cache.init_app(app)


# 设置调试程序

这里为了对比测试，我们可以在开发环境中安装[Flask-DebugToolbar](https://pypi.org/project/Flask-DebugToolbar/)来直观的监测Redis后端缓存的状态

安装Flask-DebugToolbar:
    
    $ pipenv install flask-debugtoolbar --dev

> 这里我们看到资源消耗最高的，我们可以使用Celery异步任务队列来优化，这里目前没太多要求，刚接触Celery没多久，Celery这个坑就留以后讨论吧。
实例化并初始化：

    from flask import Flask
    from flask_debugtoolbar import DebugToolbarExtension
    
    app = Flask(__name__)
    
    app.debug = True
    #意仅在debug模式启用
    
    app.config['SECRET_KEY'] = '<replace with a secret key>'
    #设置以开启cookies
    
    toolbar = DebugToolbarExtension(app)

此时设置app.debug = False就可以关闭DebugToolbar了
运行时点击右侧的FDT即可开启Flask-DebugToolbar侧栏，
![click](https://raw.githubusercontent.com/hallucigenia/grimalog/master/content/code/flask-caching-with-redis/01.png  "click")

另外，如果要载入性能分析器 可以点击Profiler的勾，勾选后再次刷新即可看到加载时间，你还可以点击查看具体函数的运行性能，以更好的优化代码。

**请最好不要在生产环境启用**，另外更方便设置环境变量可使用`flask-dotenv`



# 缓存视图函数
一般视图函数缓存优先考虑计算较多，调用数据库频繁的视图函数。并且更新不是很频繁，对我们的Blog来说，一般需要缓存的主要还是首页，除了缓存主页外，另外缓存下可能点击多的视图函数，以主页为例，我的主页里放了文章预览，在刷新后载入还是比较慢的。


    @blog_bp.route('/')
    @cache.cached(timeout= 20 * 60, unless=is_login)
    def index():
        time.sleep(1)  #根据函数复杂度可以调整或取消
        # ...
        return render_template('blog/index.html')

这里我们需要设置一个20分钟过期时间，毕竟我除了自己应该不会在我博客停留这么久，这里如果不是很常改变的界面，可以设置一个小时乃至更久。
我们可以传入一个`key_prefix`做缓存键，这样的好处在于对Redis可以使用cache.clear()方法完全清除缓存。
其他界面也可以设置缓存，比如我把个人介绍页就设置了二小时，文章页设置了30分钟，我们可以在本地测试一下

未设置缓存的时候，平均刷新一次主页需要27ms左右，实际的博客会更长
 ![未缓存的载入时间](https://raw.githubusercontent.com/hallucigenia/grimalog/master/content/code/flask-caching-with-redis/02.png  "uncaching")
而在部署完Redis缓存之后，近百倍的速度提升，这里我的提升还不是很明显
![after redis](https://raw.githubusercontent.com/hallucigenia/grimalog/master/content/code/flask-caching-with-redis/03.png  "after redis")


# 缓存其他函数
Flask-Caching的支持不仅仅是视图函数，你可以缓存非视图相关函数的结果，与缓存视图函数不同，如果没有传入参数`key_prefix`关键字来替换默认的cache_key的话，它将会默认使用当前请求的`request.path`值作为cache_key，造成直接覆写视图函数数据的情况，这里**务必使用`key_prefix`
**面对复杂的函数可以继续设置一个`time.sleep()`以模拟估算运算时间。
我还没有找到合适的应用地方，这里以官方文档给的实例代码为例：

    @cache.cached(key_prefix='MyCachedList')
    def get_list():
        return [random.randrange(0, 1) for i in range(50000)]
    my_list = get_list()

#memoize()装饰器
@cache.memoize()装饰器与@cache.cached()用法类似，函数的参数也包含在cache_key中：不同的是 @cache.memoize() 装饰器只会在再次调用相同参数时调用缓存，于是可以在函数接收到不同的实参后返回不同的结果.。 @cache.memoize() 不仅仅会缓存运行的结果, 还缓存调用时的参数值。

---


#更新缓存
以上的缓存都是计时清理的模式，但是对于博客来说，登陆的时候以及退出登录时如果没有清理缓存将会使显示错误，
cache.delete()方法可以删除缓存，其参数key_prefix默认为 `‘view/%(request.path)s’`，我们使用url_for()定位的函数键替换他以删除缓存

*auth.py:*

    @auth_bp.route('/login', methods=['GET', 'POST'])
    def login():
        cache.delete('view/%s' % url_for('blog.index'))
        # ...
        return render_template('auth/login.html', form=form)


    @auth_bp.route('/logout')
    @login_required
    def logout():
        cache.delete('view/%s' % url_for('blog.index'))
        logout_user()
        flash('Logout success.', 'info')
        return redirect_back()

相似的，如果我们新建了文章或者编辑了文章，为便于自己即时预览新更改的内容，可以在新建编辑删除文章的地方调用cache.delete()，例如：

*admin.py :*

    @admin_bp.route('/post/<int:post_id>/delete', methods=['POST'])
    @login_required
    def delete_post(post_id):
        cache.delete('view/%s' % url_for('blog.index'))
        post = Post.query.get_or_404(post_id)
        db.session.delete(post)
        db.session.commit()
        flash('Post deleted.', 'success')
        return redirect_back()

在我们删除文章的时候，通过构建缓存键同步清除首页缓存，再运行博客程序，就不会遇到删除后文章还在的问题了。
另外在编辑文章的视图函数，需要清除首页和文章页两份的缓存

 `memoize()`装饰器的缓存对应的方法为`delete_memorized()`使用方法相似。

 还存在一种`cache.clear()`的方法，可以实现清除缓存（某些后端实现并不支持）。对于我们的redis来说，倘若在其他地方也调用了Redis数据的话，一定要确保使用`key_prefix`，否则`cache.clear()`将刷新整个数据库。

 其他大部分问题都可以详见[Flask-Caching的文档](https://pythonhosted.org/Flask-Caching/)开发者写的很详细，目前我也在空闲时在Sphinx翻译文档，如果这边合适的话我会多做做贡献吧。

 有错漏之处，欢迎指正

 > 按：本计划好好做下微信公众号后端的，实在抱歉微信公众号平台那边没提前熟悉，没想微信开放的权限束手束脚，尤其是对于个人公众号，连认证的机会都没有。但相关的项目我不会抛弃，未来如果有机会申请到开放我还是会继续开发的，如果实在没有机会，我会转到`Telegram`Robot继续实现，不必在一棵假槟榔树上吊死。
