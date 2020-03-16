---
title: "Openpyxl使用 —— Python操作Excel文件總結"
date: 2020-01-05T17:32:24+08:00

categories: ['Code']
tags: ['Openpyxl', 'Excel', 'Python']
author: "Fancy"
---
本文將基本操作結合工作中常用的代碼塊提煉作為進階，期望能帶來更多啟發

<!--more-->

> 近期經常在微信發現推送的通過學習Python處理表格早點下班種種，推給主Python程序員只能說這個推薦算法還是差點意思，在日常工作當中，無論是讀取Excel試算表抑或生成數據報告，基本每隔半個月就要寫個相關代碼，有必要總結一下並在有補充的時候更新

- Q&A：為何不是`xlrd`與`xlwt`？
  - 在一開始，我的常用庫是`xlrd`與`xlwt`，直到遇到xlwt一個大量數據生成不了報告的BUG，係xls格式超過65535行則會報錯，而openpyxl支持xlsx，其實兩者差不多意思，xlrd和xlwt相較openpyxl快一點，如果寫入行數可控或對速度敏感的話，我會建議使用xlrd和xlwt，或者xlrd讀openpyxl寫。

```bash
$ pipenv install openpyxl
```

## openpyxl 讀取

```python
import openpyxl
```
引入Workbook類，並初始化。Workbook實際上對應的就是工作簿


```python
workbook = openpyxl.Workbook()
```

定位到工作表（Worksheet）

```python
sheet = workbook.active
```

或者創建sheet

```python
sheet = workbook.create_sheet(“sheet_name”) # 指定名稱
sheet = workbook.create_sheet(“sheet_name”， 1) # 指定索引位置
```
獲取sheet

```python
sheets = workbook.get_sheet_names() #獲取所有worksheet的名稱,返回可迭代對象
sheet_first = sheets[0]

sheet = workbook.get_sheet_by_name(sheet_first) #通過名稱獲取worksheet
```

刪除表

```python
sheet = workbook['sheet_name']
workbook.remove(sheet)
```



重命名表名，默認為sheet1 sheet2 。。。

```python
sheet.title = "My New Title"
```

獲取行列
```python
rows = sheet.rows # 行，返回可迭代對象
columns = sheet.columns # 列，返回可迭代對象

fat = sheet.max_row # 获取表最大工作行数
high = sheet.max_column # 获取表最大工作列数

for i in sheet["C"]:   # 獲取列
    print(i.value, ' ')
    
for i in sheet["1"]:   # 獲取行
    print(i.value, ' ')

for row in rows:
    line = [col.value for col in row]
    print(line) # 獲取行
```

通过坐标獲取單元格

```python
c = sheet.cell(5, 2) # 推薦使用
c = sheet.cell(row = 5, column = 2)
c = sheet.cell('E2')
c = sheet["E2"]
```

單元格樣式

```python
 sheet.merge_cells('A1:A4') # 合併單元格
 sheet.unmerge_cells('A1:A4') # 分解單元格
```



常用的更新單元格方式

```python
sheet.cell(1, 1).value = sheet_name # 變量
sheet.cell(1, 1).value = datetime.datetime.now() # datatime時間
sheet['E2'] = '=SUM(A1:E1)' # 公式


for row in range(1,12):
    for col in range (1,6):
        sheet.cell(row=row, column=col).value = get_column_letter(col) # OOP传递位置数据
        

list1 = ["abandon","apple", ... "zygote"] # 行內容的list數據
sheet.append(list1) # 插入一整行

line1 = [[Apple, Banana, Cherry], [Ant, Bee, Cicada], [Andy, Bill, Carmen]] #分組的數據
for index_row, line in enumerate(line1, start=1):
    for index_col, i in enumerate(line, start=1):
        sheet.cell(index_row + 1, index_col).value = i # 嵌套批量追加list数据
```



## openpyxl 寫入

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



## openpyxl 導入

打開工作簿

```python
from openpyxl import load_workbook

workbook2 = load_workbook('test_report.xlsx')
print workbook2.get_sheet_names() # >>> ['My New Title', 'Sheet'Sheet3']
```



查看更多：[官方教程](https://openpyxl.readthedocs.io/en/stable/index.html)