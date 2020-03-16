---
title: "Openpyxl使用 —— Python操作Excel文件总结"
date: 2020-02-12T17:32:24+08:00

categories: ['Code']
tags: ['Openpyxl', 'Excel', 'Python']
author: "Fancy"
---
本文将基本操作结合工作中常用的代码块提炼作为进阶，期望能带来更多启发

<!--more-->

> 近期经常在微信发现推送的通过学习Python处理表格早点下班种种，推给主Python程序员只能说这个推荐算法还是差点意思，在日常工作当中，无论是读取Excel试算表抑或生成数据报告，基本每隔半个月就要写个相关代码，有必要总结一下并在有补充的时候更新

- Q&A：为何不是`xlrd`与`xlwt`？
  - 在一开始，我的常用库是`xlrd`与`xlwt`，直到遇到xlwt一个大量数据生成不了报告的BUG，係xls格式超过65535行则会报错，而openpyxl支持xlsx，其实两者差不多意思，xlrd和xlwt相较openpyxl快一点，如果写入行数可控或对速度敏感的话，我会建议使用xlrd和xlwt，或者xlrd读openpyxl写。

```bash
$ pipenv install openpyxl
```

## openpyxl 读取

```python
import openpyxl
```
引入Workbook类，并初始化。Workbook实际上对应的就是工作簿


```python
workbook = openpyxl.Workbook()
```

定位到工作表（Worksheet）

```python
sheet = workbook.active
```

或者创建sheet

```python
sheet = workbook.create_sheet(“sheet_name”) # 指定名称
sheet = workbook.create_sheet(“sheet_name”， 1) # 指定索引位置
```
获取sheet

```python
sheets = workbook.get_sheet_names() #获取所有worksheet的名称,返回可迭代对象
sheet_first = sheets[0]

sheet = workbook.get_sheet_by_name(sheet_first) #通过名称获取worksheet
```

删除表

```python
sheet = workbook['sheet_name']
workbook.remove(sheet)
```



重命名表名，默认为sheet1 sheet2 。。。

```python
sheet.title = "My New Title"
```

获取行列
```python
rows = sheet.rows # 行，返回可迭代对象
columns = sheet.columns # 列，返回可迭代对象

fat = sheet.max_row # 获取表最大工作行数
high = sheet.max_column # 获取表最大工作列数

for i in sheet["C"]:   # 获取列
    print(i.value, ' ')
    
for i in sheet["1"]:   # 获取行
    print(i.value, ' ')

for row in rows:
    line = [col.value for col in row]
    print(line) # 获取行
```

通过坐标获取单元格

```python
c = sheet.cell(5, 2) # 推荐使用
c = sheet.cell(row = 5, column = 2)
c = sheet.cell('E2')
c = sheet["E2"]
```

单元格样式

```python
 sheet.merge_cells('A1:A4') # 合併单元格
 sheet.unmerge_cells('A1:A4') # 分解单元格
```



常用的更新单元格方式

```python
sheet.cell(1, 1).value = sheet_name # 变量
sheet.cell(1, 1).value = datetime.datetime.now() # datatime时间
sheet['E2'] = '=SUM(A1:E1)' # 公式


for row in range(1,12):
    for col in range (1,6):
        sheet.cell(row=row, column=col).value = get_column_letter(col) # OOP传递位置数据
        

list1 = ["abandon","apple", ... "zygote"] # 行内容的list数据
sheet.append(list1) # 插入一整行

line1 = [[Apple, Banana, Cherry], [Ant, Bee, Cicada], [Andy, Bill, Carmen]] #分组的数据
for index_row, line in enumerate(line1, start=1):
    for index_col, i in enumerate(line, start=1):
        sheet.cell(index_row + 1, index_col).value = i # 嵌套批量追加list数据
```



## openpyxl 写入

保存工作簿（文本方式）

```python
workbook.save('test_report.xlsx')
```

保存工作簿（文件流方式）

```python
from io import BytesIO

file_stream = io.BytesIO()
workbook.save(file_stream)
file_stream.seek(0)

with open("test_report.xlsx", "wb") as fp:
  fp.write(file_stream.read())
```



## openpyxl 导入

打开工作簿

```python
from openpyxl import load_workbook

workbook2 = load_workbook('test_report.xlsx')
print workbook2.get_sheet_names() # >>> ['My New Title', 'Sheet'Sheet3']
```



查看更多：[官方教程](https://openpyxl.readthedocs.io/en/stable/index.html)