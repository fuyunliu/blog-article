---
title: Python 处理 csv 文件常见错误
date: 2018-12-03 10:53:31
categories:
- 笔记
tags:
- Python3
---

在用 Python 处理 csv 文件时遇到2个错误，记录下处理方法。

<!-- more -->

<!-- toc -->

## 字段包含 `NULL` 值

csv 文件中字段包含 NULL 值会出错，解决方法是读取文件时把 NULL 值替换为空字符串。

```python
import csv

with open('test.csv', 'rt', encoding='utf-8') as f:
    fc = csv.DictReader((line.replace('\0', '') for line in f))
    # do something with fc
```

## `OverflowError and maxInt`

```python
import csv
import sys

maxInt = sys.maxsize
decrement = True

while decrement:
    # decrease the maxInt value by factor 10
    # as long as the OverflowError occurs.

    decrement = False
    try:
        csv.field_size_limit(maxInt)
    except OverflowError:
        maxInt = int(maxInt / 10)
        decrement = True

with open('test.csv', 'rt', encoding='utf-8') as f:
    fc = csv.DictReader(f)
    # do something with fc

```
