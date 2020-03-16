---
title: "Python标准库 —— pdb调试工具使用"
date: 2020-02-20T15:40:01+08:00
categories: ['Code']
tags: ['Debug', 'Python']
author: "Fancy"

resizeImages: false
---

日常DEBUG中，我常用traceback以及Pycharm的无脑断点调试，直到有一天线上在高并发下，僵尸进程暴涨导致卡死，无法精准的定位某些异常时，掌握Python3自带的pdb标准库或ipdb进行单步调试就显得十分必要。
<!--more-->

- Q&A：为何不是Pycharm的远程调试？
   - 当时的我只用了Pycharm SFTP上传代码和本地调试，至于远程Debug，就像一开始从Vim转到Pycharm一样，感觉多学点积累并不是坏事，后来自己在空闲的时候补习实践了更多Pycharm的进阶，后期进行Pycharm相关的积累记录时会进行总结比较，熟练之后，pdb还是很高效的一种debug方式。

- ipdb
  - pdb时Python标准库提供的调试器，而ipdb则需要pip install，基本功能相同，相当于Ipython + pdb，增强型的pdb多了代码补全以及语法高亮种种。

### PDB全部 Debugger 命令

重点关注加粗部分即可，有需要再来速查

| 命令               | 解释                                                         |
| :----------------- | :----------------------------------------------------------- |
| **h(elp)**         | 帮助                                                         |
| **w(here)**        | 打印当前堆栈跟踪位置                                         |
| d(own)             | 执行跳转到在当前堆栈的下一层                                 |
| u(p)               | 执行跳转到当前堆栈的上一层                                   |
| **b(reak)**        | 设置断点,line_no：当前脚本的line_no行添加断点        filename:line_no：脚本filename的line_no行添加断点         function：在函数function的第一条可执行语句处添加断点 |
| **tbreak**         | 临时断点，运行一次即删除，参数与 *break* 同                  |
| **cl(ear)**        | 清除断点，默认清除所有断点     bpnumber1 bpnumber2... 清除断点号为bpnumber1,bpnumber2...的断点       lineno 清除当前脚本lineno行的断点      filename:line_no 清除脚本filename的line_no行的断点 |
| **disable/enable** | 停用但不清除/激活断点                                        |
| ignore             | 忽略断点                                                     |
| condition          | 设置条件断点                                                 |
| commands           | 指定断点号 *bpnumber* 的命令                                 |
| **s(tep)**         | 继续执行，进入函数或当前函数                                 |
| **n(ext)**         | 继续执行，当前函数下一行                                     |
| **unt(il)**        | 默认执行到更大一个行数（跳出循环），支持指定行数             |
| **r(eturn)**       | 执行代码直到从当前函数返回                                   |
| **c(ont(inue))**   | 继续执行程序直到下一条断点                                   |
| **j(ump)**         | 跳转到指定的行数                                             |
| **l(ist)**         | 查看当前行的代码段                                           |
| ll                 | 列出所有代码，行标记为列表                                   |
| **a(rgs)**         | 打印当前函数的参数                                           |
| **p**              | 打印变量（expression）的值                                   |
| **pp**             | 以一种更漂亮的方式打印变量（expression）的值                 |
| whatis             | 打印变量（expression）的类型                                 |
| source             | 显示尝试获取的源代码                                         |
| display            | 显示已更改的表达式的值                                       |
| undisplay          | 不显示已更改的表达式的值                                     |
| interact           | 启动交互式解释器                                             |
| alias              | 创建执行命令的别名                                           |
| unalias            | 删除别名                                                     |
| !                  | 后面跟单行语句，可以直接在当前堆栈框架执行                   |
| **run**            | 重新运行程序                                                 |
| restart            | run的别名                                                    |
| **q(uit)**         | 中止并退出                                                   |
| debug              | 输入递归调试器，用于遍历参数。                               |
| retval             | 打印函数最后一次返回值                                       |

### 根据应用场景选择不同的方式：

- pdb启动Python文件

```bash
$ python3 -m pdb myscript.py
```

- 内部调用

```python
# !usr/bin/python
# -*- coding: utf-8 -*-

import pdb
import time


def countit(n):
    s=0
    for i in range(n):
        time.sleep(1)
        pdb.set_trace()
        print(s)
        s += i
        
if __name__ == "__main__":
    seconds = 10
    countit(seconds)
```



```python
$ python3 myscript.py
```

两种方式没有实际上的区别，根据文件大小以及应用环境灵活运用即可。

启动调试器后，如第一种方式pdb会在每一步停止，第二种则在指定的断点处停止，pdb显示断点下一行，便可以使用上部表格提供的命令，进行断点调试

```bash
$ python3 -m pdb test.py
> /Users/dilophosaurus/PycharmProjects/untitled1/test.py(1)<module>()
-> print(s)
(Pdb)
```

### 堆栈跟踪

```bash
$ python3 myscript.py
> /Users/fancy/test_pdb.py(13)countit()
-> print(s)
(Pdb) w         # 显示当前行及堆栈位置
  /Users/fancy/test_pdb.py(18)<module>()
-> countit(seconds)
> /Users/fancy/test_pdb.py(13)countit()
-> print(s)
(Pdb) l         # 显示代码上下文
  8  	def countit(n):
  9  	    s=0
 10  	    for i in range(n):
 11  	        time.sleep(1)
 12  	        pdb.set_trace()
 13  ->	        print(s)
 14  	        s += i
 15
 16  	if __name__ == "__main__":
 17  	    seconds = 10
 18  	    countit(seconds)
(Pdb) n         # 向下单步执行
0
> /Users/fancy/test_pdb.py(14)countit()
-> s += i
(Pdb) up        # 向上调用堆栈
> /Users/fancy/test_pdb.py(18)<module>()
-> countit(seconds)
(Pdb) down      # 向下调用堆栈
> /Users/fancy/test_pdb.py(14)countit()
-> s += i
(Pdb) c         # 继续执行程序直到下一条断点  
> /Users/dilophosaurus/PycharmProjects/untitled1/test_pdb.py(12)countit()
-> pdb.set_trace()
(Pdb) s         # 继续执行，区别于n（可见会进入调用函数内部）
--Call--
> /usr/local/Cellar/python/3.7.6_1/Frameworks/Python.framework/Versions/3.7/lib/python3.7/pdb.py(1604)set_trace()
-> def set_trace(*, header=None):
(Pdb) a        # 打印**当前**函数的参数
header = None 
(Pdb) c
0
> /Users/dilophosaurus/PycharmProjects/untitled1/test_pdb.py(13)countit()
-> print(s)
(Pdb) pp s     # `s`作为pdb保留关键字（step），必须用过`p`或者`pp`进行显示
1
(Pdb) h        # 回顾下我们所使用的命令

Documented commands (type help <topic>):
========================================
EOF    c          d        h         list      q        rv       undisplay
a      cl         debug    help      ll        quit     s        unt
alias  clear      disable  ignore    longlist  r        source   until
args   commands   display  interact  n         restart  step     up
b      condition  down     j         next      return   tbreak   w
break  cont       enable   jump      p         retval   u        whatis
bt     continue   exit     l         pp        run      unalias  where

Miscellaneous help topics:
==========================
exec  pdb

(Pdb)
```

更多请参考[官方文档](https://docs.python.org/3/library/pdb.html)

