---
title: "利用Redis做Flask緩存後端"
date: 2019-03-01T17:22:21+08:00

categories: ['Code']
tags: ['Flask', 'Python', 'Redis']
author: "Fancy"
draft: false
resizeImages: false

---

Flask項目中使用Flask-Caching實現Redis緩存
<!--more-->

> NoSQL數據庫中的Redis，其key-value支持的value類型相當豐富，數據庫完全保存在內存裏，支持使用硬盤持久化，甚至為數據結構服務器。基於Redis集群原理後期還可以做分布式緩存

# 使用 Flask-Caching

在這裏我們調用[Flask-Caching](https://github.com/thadeusb/flask-cache)以更加便利的實現Redis的後端緩存。這次就以本Blog為例，典型化介紹Redis緩存的實現。
Flask-Caching基本的功能繼承於flask-cache，大致可以分為

- 記憶參數型緩存：由@cached裝飾器實現 
- 無記憶參數型緩存：由@memoize裝飾器實現

*Flask-Caching是Flask-Cache的fork版本，原來的Flask-Cache已經四年沒更新了，最後壹個版本只支持到python 3.3*


    pipenv install flask-caching

如果沒有安裝redis必須的
    

    pipenv install redis

工廠函數裏實例化Cache類，創建壹個cache對象，如果沒實例化redis也是必須的(這裏我把拓展工廠函數分開了，根據自身情況做相應更改即可)：
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

這裏只是實例，Flask-Caching除了調用redis緩存後端之外還可以使用本地Python字典，uwsgi的緩存框架，mamcached等等，詳見[官方文檔](https://pythonhosted.org/Flask-Caching/#configuring-flask-caching)

其中`RedisCache`有以下字段可用：

| CACHE_TYPE': 'redis'  | Flask-Caching調用redis緩存後端必須的字段                     |
| :-------------------- | :----------------------------------------------------------- |
| CACHE_DEFAULT_TIMEOUT | 未指定緩存超時時間時的默認緩存超時時間，默認為300秒          |
| CACHE_KEY_PREFIX      | 在所有鍵之前添加的前綴。讓不同的應用程序可以使用相同的服務器。 |
| CACHE_REDIS_HOST      | Redis的服務主機，未指定則為`localhost`即`127.0.0.1`          |
| CACHE_REDIS_PORT      | Redis的服務端口，默認即Redis默認為`6379`                     |
| CACHE_REDIS_PASSWORD  | Redis如指定密碼時的設置                                      |
| CACHE_REDIS_DB        | Redis的db庫（基於零號索引），默認為零                        |
| CACHE_ARGS            | 在緩存類實例化的時候會對該列表進行解包拆分並傳遞。           |
| CACHE_OPTIONS         | 在緩存類實例化的時候會傳遞該字典                             |
| CACHE_REDIS_URL       | 連接到Redis服務器的URL，如：`redis://user:password@localhost:6379/2. `相當於把前面的幾點整合壹下， |

傳入參數並初始化拓展，註冊到app對象：
*__init__.py :*

    from cryptic.extensions import cache
    # ...
    def register_extensions(app):
        # ...
        cache.init_app(app)


# 設置調試程序

這裏為了對比測試，我們可以在開發環境中安裝[Flask-DebugToolbar](https://pypi.org/project/Flask-DebugToolbar/)來直觀的監測Redis後端緩存的狀態

安裝Flask-DebugToolbar:
    

    $ pipenv install flask-debugtoolbar --dev

> 這裏我們看到資源消耗最高的，我們可以使用Celery異步任務隊列來優化，這裏目前沒太多要求，剛接觸Celery沒多久，Celery這個坑就留以後討論吧。
> 實例化並初始化：

    from flask import Flask
    from flask_debugtoolbar import DebugToolbarExtension
    
    app = Flask(__name__)
    
    app.debug = True
    #意僅在debug模式啟用
    
    app.config['SECRET_KEY'] = '<replace with a secret key>'
    #設置以開啟cookies
    
    toolbar = DebugToolbarExtension(app)

此時設置app.debug = False就可以關閉DebugToolbar了
運行時點擊右側的FDT即可開啟Flask-DebugToolbar側欄，
![click](https://raw.githubusercontent.com/hallucigenia/grimalog/master/content/code/flask-caching-with-redis/01.png  "click")

另外，如果要載入性能分析器 可以點擊Profiler的勾，勾選後再次刷新即可看到加載時間，妳還可以點擊查看具體函數的運行性能，以更好的優化代碼。

**請最好不要在生產環境啟用**，另外更方便設置環境變量可使用`flask-dotenv`



# 緩存視圖函數

壹般視圖函數緩存優先考慮計算較多，調用數據庫頻繁的視圖函數。並且更新不是很頻繁，對我們的Blog來說，壹般需要緩存的主要還是首頁，除了緩存主頁外，另外緩存下可能點擊多的視圖函數，以主頁為例，我的主頁裏放了文章預覽，在刷新後載入還是比較慢的。


    @blog_bp.route('/')
    @cache.cached(timeout= 20 * 60, unless=is_login)
    def index():
        time.sleep(1)  #根據函數復雜度可以調整或取消
        # ...
        return render_template('blog/index.html')

這裏我們需要設置壹個20分鐘過期時間，畢竟我除了自己應該不會在我博客停留這麽久，這裏如果不是很常改變的界面，可以設置壹個小時乃至更久。
我們可以傳入壹個`key_prefix`做緩存鍵，這樣的好處在於對Redis可以使用cache.clear()方法完全清除緩存。
其他界面也可以設置緩存，比如我把個人介紹頁就設置了二小時，文章頁設置了30分鐘，我們可以在本地測試壹下

未設置緩存的時候，平均刷新壹次主頁需要27ms左右，實際的博客會更長
 ![未緩存的載入時間](https://raw.githubusercontent.com/hallucigenia/grimalog/master/content/code/flask-caching-with-redis/02.png  "uncaching")
而在部署完Redis緩存之後，近百倍的速度提升，這裏我的提升還不是很明顯
![after redis](https://raw.githubusercontent.com/hallucigenia/grimalog/master/content/code/flask-caching-with-redis/03.png  "after redis")


# 緩存其他函數

Flask-Caching的支持不僅僅是視圖函數，妳可以緩存非視圖相關函數的結果，與緩存視圖函數不同，如果沒有傳入參數`key_prefix`關鍵字來替換默認的cache_key的話，它將會默認使用當前請求的`request.path`值作為cache_key，造成直接覆寫視圖函數數據的情況，這裏**務必使用`key_prefix`

**面對復雜的函數可以繼續設置壹個`time.sleep()`以模擬估算運算時間。
我還沒有找到合適的應用地方，這裏以官方文檔給的實例代碼為例：

    @cache.cached(key_prefix='MyCachedList')
    def get_list():
        return [random.randrange(0, 1) for i in range(50000)]
    my_list = get_list()

#memoize()裝飾器

@cache.memoize()裝飾器與@cache.cached()用法類似，函數的參數也包含在cache_key中：不同的是 @cache.memoize() 裝飾器只會在再次調用相同參數時調用緩存，於是可以在函數接收到不同的實參後返回不同的結果.。 @cache.memoize() 不僅僅會緩存運行的結果, 還緩存調用時的參數值。

---


#更新緩存

以上的緩存都是計時清理的模式，但是對於博客來說，登陸的時候以及退出登錄時如果沒有清理緩存將會使顯示錯誤，
cache.delete()方法可以刪除緩存，其參數key_prefix默認為 `‘view/%(request.path)s’`，我們使用url_for()定位的函數鍵替換他以刪除緩存

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

相似的，如果我們新建了文章或者編輯了文章，為便於自己即時預覽新更改的內容，可以在新建編輯刪除文章的地方調用cache.delete()，例如：

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

在我們刪除文章的時候，通過構建緩存鍵同步清除首頁緩存，再運行博客程序，就不會遇到刪除後文章還在的問題了。
另外在編輯文章的視圖函數，需要清除首頁和文章頁兩份的緩存

 `memoize()`裝飾器的緩存對應的方法為`delete_memorized()`使用方法相似。

 還存在壹種`cache.clear()`的方法，可以實現清除緩存（某些後端實現並不支持）。對於我們的redis來說，倘若在其他地方也調用了Redis數據的話，壹定要確保使用`key_prefix`，否則`cache.clear()`將刷新整個數據庫。

 其他大部分問題都可以詳見[Flask-Caching的文檔](https://pythonhosted.org/Flask-Caching/)開發者寫的很詳細，目前我也在空閑時在Sphinx翻譯文檔，如果這邊合適的話我會多做做貢獻吧。

 有錯漏之處，歡迎指正

 > 按：本計劃好好做下微信公眾號後端的，實在抱歉微信公眾號平臺那邊沒提前熟悉，沒想微信開放的權限束手束腳，尤其是對於個人公眾號，連認證的機會都沒有。但相關的項目我不會拋棄，未來如果有機會申請到開放我還是會繼續開發的，如果實在沒有機會，我會轉到`Telegram`Robot繼續實現，不必在壹棵假檳榔樹上吊死。