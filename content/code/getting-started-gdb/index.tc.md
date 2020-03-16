---
title: "Python標準庫 —— pdb調試工具使用"
date: 2020-02-20T15:40:01+08:00
categories: ['Code']
tags: ['Debug', 'Python']
author: "Fancy"

resizeImages: false
---

日常DEBUG中，我常用traceback以及Pycharm的無腦斷點調試，直到有壹天線上在高並發下，僵屍進程暴漲導致卡死，無法精準的定位某些異常時，掌握Python3自帶的pdb標準庫或ipdb進行單步調試就顯得十分必要。
<!--more-->

- Q&A：為何不是Pycharm的遠程調試？
  - 當時的我只用了Pycharm SFTP上傳代碼和本地調試，至於遠程Debug，就像壹開始從Vim轉到Pycharm壹樣，感覺多學點積累並不是壞事，後來自己在空閑的時候補習實踐了更多Pycharm的進階，後期進行Pycharm相關的積累記錄時會進行總結比較，熟練之後，pdb還是很高效的壹種debug方式。

- ipdb
  - pdb時Python標準庫提供的調試器，而ipdb則需要pip install，基本功能相同，相當於Ipython + pdb，增強型的pdb多了代碼補全以及語法高亮種種。

### PDB全部 Debugger 命令

重點關註加粗部分即可，有需要再來速查

| 命令               | 解釋                                                         |
| :----------------- | :----------------------------------------------------------- |
| **h(elp)**         | 幫助                                                         |
| **w(here)**        | 打印當前堆棧跟蹤位置                                         |
| d(own)             | 執行跳轉到在當前堆棧的下壹層                                 |
| u(p)               | 執行跳轉到當前堆棧的上壹層                                   |
| **b(reak)**        | 設置斷點，line_no：當前腳本的line_no行添加斷點        filename:line_no：腳本filename的line_no行添加斷點         function：在函數function的第壹條可執行語句處添加斷點 |
| **tbreak**         | 臨時斷點，運行壹次即刪除，參數與 *break* 同                  |
| **cl(ear)**        | 清除斷點，默認清除所有斷點     bpnumber1 bpnumber2... 清除斷點號為bpnumber1,bpnumber2...的斷點       lineno 清除當前腳本lineno行的斷點      filename:line_no 清除腳本filename的line_no行的斷點 |
| **disable/enable** | 停用但不清除/激活斷點                                        |
| ignore             | 忽略斷點                                                     |
| condition          | 設置條件斷點                                                 |
| commands           | 指定斷點號 *bpnumber* 的命令                                 |
| **s(tep)**         | 繼續執行，進入函數或當前函數                                 |
| **n(ext)**         | 繼續執行，當前函數下壹行                                     |
| **unt(il)**        | 默認執行到更大壹個行數（跳出循環），支持指定行數             |
| **r(eturn)**       | 執行代碼直到從當前函數返回                                   |
| **c(ont(inue))**   | 繼續執行程序直到下壹條斷點                                   |
| **j(ump)**         | 跳轉到指定的行數                                             |
| **l(ist)**         | 查看當前行的代碼段                                           |
| ll                 | 列出所有代碼，行標記為列表                                   |
| **a(rgs)**         | 打印當前函數的參數                                           |
| **p**              | 打印變量（expression）的值                                   |
| **pp**             | 以壹種更漂亮的方式打印變量（expression）的值                 |
| whatis             | 打印變量（expression）的類型                                 |
| source             | 顯示嘗試獲取的源代碼                                         |
| display            | 顯示已更改的表達式的值                                       |
| undisplay          | 不顯示已更改的表達式的值                                     |
| interact           | 啟動交互式解釋器                                             |
| alias              | 創建執行命令的別名                                           |
| unalias            | 刪除別名                                                     |
| !                  | 後面跟單行語句，可以直接在當前堆棧框架執行                   |
| **run**            | 重新運行程序                                                 |
| restart            | run的別名                                                    |
| **q(uit)**         | 中止並退出                                                   |
| debug              | 輸入遞歸調試器，用於遍歷參數。                               |
| retval             | 打印函數最後壹次返回值                                       |

### 根據應用場景選擇不同的方式：

- pdb啟動Python文件

```bash
$ python3 -m pdb myscript.py
```

- 內部調用

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

兩種方式沒有實際上的區別，根據文件大小以及應用環境靈活運用即可。

啟動調試器後，如第壹種方式pdb會在每壹步停止，第二種則在指定的斷點處停止，pdb顯示斷點下壹行，便可以使用上部表格提供的命令，進行斷點調試

```bash
$ python3 -m pdb test.py
> /Users/dilophosaurus/PycharmProjects/untitled1/test.py(1)<module>()
-> print(s)
(Pdb)
```

### 堆棧跟蹤

```bash
$ python3 myscript.py
> /Users/fancy/test_pdb.py(13)countit()
-> print(s)
(Pdb) w         # 顯示當前行及堆棧位置
  /Users/fancy/test_pdb.py(18)<module>()
-> countit(seconds)
> /Users/fancy/test_pdb.py(13)countit()
-> print(s)
(Pdb) l         # 顯示代碼上下文
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
(Pdb) n         # 向下單步執行
0
> /Users/fancy/test_pdb.py(14)countit()
-> s += i
(Pdb) up        # 向上調用堆棧
> /Users/fancy/test_pdb.py(18)<module>()
-> countit(seconds)
(Pdb) down      # 向下調用堆棧
> /Users/fancy/test_pdb.py(14)countit()
-> s += i
(Pdb) c         # 繼續執行程序直到下壹條斷點  
> /Users/dilophosaurus/PycharmProjects/untitled1/test_pdb.py(12)countit()
-> pdb.set_trace()
(Pdb) s         # 繼續執行，區別於n（可見會進入調用函數內部）
--Call--
> /usr/local/Cellar/python/3.7.6_1/Frameworks/Python.framework/Versions/3.7/lib/python3.7/pdb.py(1604)set_trace()
-> def set_trace(*, header=None):
(Pdb) a        # 打印**當前**函數的參數
header = None 
(Pdb) c
0
> /Users/dilophosaurus/PycharmProjects/untitled1/test_pdb.py(13)countit()
-> print(s)
(Pdb) pp s     # `s`作為pdb保留關鍵字（step），必須用過`p`或者`pp`進行顯示
1
(Pdb) h        # 回顧下我們所使用的命令

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

更多請參考[官方文檔](https://docs.python.org/3/library/pdb.html)

