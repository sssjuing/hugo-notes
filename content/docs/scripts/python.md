---
title: Python脚本
---

---

## mongo 数据迁移

```python
from pymongo import MongoClient

src_url = "mongodb://mongoadmin:mongopassword@192.168.106.132:27017"
dest_url = "mongodb://mongoadmin:mongopassword@localhost:27017"
database_list = ['books_db', 'hourse_db', 'maya_corpus', 'music_db']

src_client = MongoClient(src_url)
dest_client = MongoClient(dest_url)

print('all db\'s names are', src_client.list_database_names())

for db_name in database_list:
    src_db = src_client[db_name]
    for collection_name in src_db.list_collection_names():
        count = src_db[collection_name].count_documents(filter={})
        print(db_name, collection_name, count)
        records = src_db[collection_name].find()
        print('duplicating')
        dest_client[db_name][collection_name].insert_many(records)
```

## 扫描文件夹内的全部文件

```python
import os

def walk_files(dir_path, suffix = None):
    """
    :param dir_path:str 目标文件夹的路径
    :param suffix:str 扫描指定后缀的文件, 传入None将扫描所有文件
    """
    files_list = []
    for root, sub_dirs, files in os.walk(dir_path):
        for special_file in files:
            if (True if suffix is None else special_file.endswith(suffix)):
                file_name = special_file
                file_path = os.path.join(root, special_file).replace('\\', '/')
                files_list.append({
                    'file_name': file_name,
                    'file_path': file_path,
                    'file_type': os.path.splitext(file_path)[-1],
                })
    return files_list
```

```python
from datetime import datetime, timezone, timedelta
from collections import deque
from pathlib import Path


def sec_to_rfc3339(timestamp):
    date = datetime.fromtimestamp(timestamp)
    return date.replace(tzinfo=timezone(timedelta(hours=8))).isoformat()


def scan_files(dir_path, suffix = None):
    """
    :param dir_path:str 目标文件夹的路径
    :param suffix:str 扫描指定后缀的文件, 传入None将扫描所有文件
    """
    file_list = []
    dir_queue = deque([Path(dir_path)])
    while len(dir_queue) > 0:
        folder = dir_queue.popleft()
        for entry in folder.iterdir():
            if entry.is_dir():
                dir_queue.append(entry)
            elif entry.is_file():
                if (True if suffix is None else entry.suffix == suffix):
                    info = entry.stat()
                    file_list.append({
                        'file_name': entry.name,
                        'file_path': str(entry),
                        'size': info.st_size,
                        'created_at': sec_to_rfc3339(info.st_ctime),
                        'type': entry.suffix,
                    })
    return file_list
```

## 格式化打印字典数据

```python
import json

def print_dict_data(data: dict):
    print(json.dumps(data, sort_keys=True, indent=4,
          separators=(', ', ': '), ensure_ascii=False))
```

## 转换 pdf 文件为 csv

```python
# http://zjw.beijing.gov.cn/bjjs/fwgl/index.shtml

import pdfplumber
import csv

def read_pdf():
  result = []
  with pdfplumber.open("3.pdf") as pdf:
    for page in pdf.pages:
      text = page.extract_text() #提取文本
      rows = filter(lambda item: not item.isspace(), text.split('\n'))
      for row in rows:
        cols = row.strip().split()
        if(len(cols) == 3 and cols[0].isdigit()):
          result.append(cols)
  return result

def wirte_csv(rows):
  with open('3.csv', 'w', encoding='utf-8', newline='') as f:
    writer = csv.writer(f, dialect='excel')
    writer.writerows(rows)

def main():
  result = read_pdf()
  wirte_csv(result)
  print(len(result))

if __name__ == "__main__":
  main()
```
