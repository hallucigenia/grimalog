---
title: "為 SqlAlchemy 添加分頁邏輯"
date: 2020-02-13T22:40:20+08:00

categories: ['Code']
tags: ['SQLAlchemy', 'Python', 'Code']
author: "Fancy"
noSummary: true
draft: true
resizeImages: false
---

<!--more-->

在我们谈论如何为SqlAlchemy 添加分页逻辑之前，我想先说一下Flask-SQLAlchemy在路由外的调用。Flask-SQLAlchemy是将SqlAlchemy的相关概念绑定到了Flask模型当中，为实例话还有引用提供很多便利，例如不再需要`create_engine` 包括一些Web常用的切换数据库，查询，过滤，以及映射，都有了不同程度的优化，Flask-SQLAlchemy使Flask使用ORM变得非常简单。


如果你的Flask框架中包含很多业务逻辑，比如要在blueprint之外的其他地方调用回相同的数据库（比如在蓝本里引用对象，在对象里实现ORM查询），我们尝试用正常的逻辑编写


这时却收获一个RuntimeError找不到应用上下文，使用flask sqlalchemy的时候，不仅要小心别循环引用，找不到上下文的报错如果不认真的话确实会偶遇到。


---
在用过Flask-SQLAlchemy之后，有时会将其和SQLAlchemy的方法搞混淆，曾接过这样一个需求，需要返回的RESTful API里，需要通过后端实现不同字段的排序或筛选或者兼而有之，以及分页逻辑，ORM引擎为SQLAlchemy，这个确实做起来比较坑，大量的代码非常不Python，秉着不重复造轮子的做法，我们通过阅读Flask-SQLAlchemy的源码学了一些新姿势，
```python
from math import ceil

from flask import request, abort
from flask_sqlalchemy import xrange
from sqlalchemy.orm import Query


class Pagination(object):
    """
    Sqlalchemy 的分页对象模块
    """

    def __init__(self, query, page, per_page, total, items):
        self.query = query
        self.page = page
        self.per_page = per_page
        self.total = total
        self.items = items

    @property
    def pages(self):
        if self.per_page == 0:
            pages = 0
        else:
            pages = int(ceil(self.total / float(self.per_page)))
        return pages

    def prev(self, error_out=False):
        assert self.query is not None, 'a query object is required ' \
                                       'for this method to work'
        return self.query.paginate(self.page - 1, self.per_page, error_out)

    @property
    def prev_num(self):
        if not self.has_prev:
            return None
        return self.page - 1

    @property
    def has_prev(self):
        return self.page > 1

    def next(self, error_out=False):
        assert self.query is not None, 'a query object is required ' \
                                       'for this method to work'
        return self.query.paginate(self.page + 1, self.per_page, error_out)

    @property
    def has_next(self):
        return self.page < self.pages

    @property
    def next_num(self):
        if not self.has_next:
            return None
        return self.page + 1

    def iter_pages(self, left_edge=2, left_current=2,
                   right_current=5, right_edge=2):
        last = 0
        for num in xrange(1, self.pages + 1):
            if num <= left_edge or \
               (num > self.page - left_current - 1 and
                num < self.page + right_current) or \
               num > self.pages - right_edge:
                if last + 1 != num:
                    yield None
                yield num
                last = num


def paginate(self, page=None, per_page=None, error_out=True):
    """
    分页函数
    :param self:
    :param page:
    :param per_page:
    :param error_out:
    :return: 分页对象
    """
    if request:
        if page is None:
            try:
                page = int(request.args.get('page', 1))
            except (TypeError, ValueError):
                if error_out:
                    abort(404)

                page = 1

        if per_page is None:
            try:
                per_page = int(request.args.get('per_page', 20))
            except (TypeError, ValueError):
                if error_out:
                    abort(404)

                per_page = 20
    else:
        if page is None:
            page = 1

        if per_page is None:
            per_page = 20

    if error_out and page < 1:
        abort(404)

    items = self.limit(per_page).offset((page - 1) * per_page).all()

    if not items and page != 1 and error_out:
        abort(404)

    if page == 1 and len(items) < per_page:
        total = len(items)
    else:
        total = self.order_by(None).count()

    return Pagination(self, page, per_page, total, items)


Query.paginate = paginate  # add method for SQLAlchemy from flask-SQLAlchemy
```
这样引用本模块

```python
from paginate_plugin import Query
```
使用中，我们初始化的session就可以简单的直接调用模块里定义的`paginate`分页函数, 举个栗子：

```python
...
def paginate_main(user_id, t_id, page_size, page_index, sort, order, search):
    session = Database().Session()
    try:
        per_page, page = int(page_size), int(page_index)
        db_check = Task.valid_field_attr(Task, str(sort))
        sort_func = sort if db_check else "added_on"
        order_func = desc if order == "descend" else asc
        if search:
            task_all = session.query(Task).filter(
                Task.user == user_id).filter(
                Task.task_name.like(
                    "%" +
                    search +
                    "%")).group_by(
                Task.task_id).order_by(
                order_func(sort_func)).paginate(
                    page,
                per_page)
...
    except Exception as e:
        logger.error("Exception: {0}".format(traceback.format_exc()))
        ret = APIResponse(ret_code=ECODE_UNKNOWN).get_resp()
    else:
        ret = APIResponse(ret_code=ECODE_SUCCESS, ret_data=ret_task).get_resp()
    finally:
        session.close()

    return ret
```


> Flask自由的结构组织方式带来很多便利性，根据不同的业务需求我会调整我的项目结构，但并不是每个Flask项目都如你所愿，有时在架构设计的时候，并没有办法考虑到产品后来的需求走向，版本需求迭代后会出现很多问题，这也是本文的来源之一。