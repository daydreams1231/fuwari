---
title: Openlist data.db文件异常变大.md
published: 2025-09-18
description: '解决Openlist/Alist在长时间使用后data.db文件异常变大问题'
image: ''
tags: [Alist, Openlist]
category: '教程'
draft: false 
lang: ''
---

如果你的环境没有Python, 请将data.db文件复制到任意有python环境的地方 <br>

## 打开Python终端, 依次执行如下内容
```python
import sqlite3

conn = sqlite3.connect('data.db文件位置')
cursor = conn.cursor()

cursor.execute("VACUUM")

# 关闭连接
cursor.close()
conn.close()
```