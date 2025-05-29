---
title: Python 脚本
weight: 1
---

---

### 离线安装 Python 第三方包

```bash
# (可选)，将 pip 升级到最新版
python -m pip install -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple --upgrade pip

# (可选)，设置 pip 的默认源
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple

# 将已安装的第三方包信息存储到 requirements.txt 文件
pip freeze > requirements.txt

# 将第三方包下载到当前目录下的 pkgs 目录中
pip download -d ./pkgs -r requirements.txt -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple

# 复制 pkgs 目录到离线的目标主机中, 执行以下命令离线安装第三方包
pip install --no-index --ignore-installed --find-links=./pkgs -r requirements.txt
```

### MongoDB 数据迁移

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

### 扫描文件夹内的全部文件

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

### 格式化打印字典数据

```python
import json

def print_dict_data(data: dict):
    print(json.dumps(data, sort_keys=True, indent=4,
          separators=(', ', ': '), ensure_ascii=False))
```

### 转换 pdf 文件为 csv

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

### 将 python 程序部署为 Windows 服务

```python
import win32serviceutil
import win32service
import win32event
import servicemanager
import socket
import time

class PythonService(win32serviceutil.ServiceFramework):
    _svc_name_ = 'PythonService'  # 服务名称
    _svc_display_name_ = 'Python Service'  # 服务显示名称
    _svc_description_ = 'This is a Python service.'  # 服务描述

    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.hWaitStop = win32event.CreateEvent(None, 0, 0, None)
        socket.setdefaulttimeout(60)

    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        win32event.SetEvent(self.hWaitStop)

    def SvcDoRun(self):
        servicemanager.LogMsg(servicemanager.EVENTLOG_INFORMATION_TYPE,
                              servicemanager.PYS_SERVICE_STARTED,
                              (self._svc_name_, ''))
        self.main()

    def main(self):
        # 这里添加您的服务逻辑
        while True:
            time.sleep(1)
            # 检查是否需要停止服务
            if win32event.WaitForSingleObject(self.hWaitStop, 0) == win32event.WAIT_OBJECT_0:
                break

if __name__ == '__main__':
    win32serviceutil.HandleCommandLine(PythonService)
```

### MySQL 数据迁移

```python
import pymysql

def connect_to_database(host_name, user_name, user_password, db_name):
    return pymysql.connect(
        host=host_name,
        user=user_name,
        password=user_password,
        database=db_name,
        charset='utf8mb4',
        cursorclass=pymysql.cursors.DictCursor
    )

def migrate_data_with_transformations(source_db, target_db, source_table, target_table):
    source_conn = connect_to_database('source_ip', 'source_username', 'source_password', source_db)
    target_conn = connect_to_database('target_ip', 'target_username', 'target_password', target_db)

    try:
        with source_conn.cursor() as source_cursor:
            # 获取源表的所有字段
            source_cursor.execute(f"SHOW COLUMNS FROM {source_table}")
            columns = [col['Field'] for col in source_cursor.fetchall()]

            # 根据映射关系和目标表的字段顺序构造SELECT语句
            select_fields = ', '.join(columns)
            source_cursor.execute(f"SELECT {select_fields} FROM {source_table}")
            rows = source_cursor.fetchall()
            # print(select_fields)

        with target_conn.cursor() as target_cursor:
            for row in rows:
                # 字段转换
                row['cover_path'] = row['cover_path'][11:]
                bucket_path = row.pop('bucket_path')
                row['video_path'] = bucket_path and bucket_path[11:]

                # 根据映射关系和目标表的字段顺序构造INSERT语句的字段和值
                target_fields = ', '.join(row.keys())
                target_values = ', '.join(['%s'] * len(row))
                sql = f"INSERT INTO {target_table} ({target_fields}) VALUES ({target_values})"
                print(row)
                print(sql)
                print(row.values())
                target_cursor.execute(sql, tuple(row.values()))

        target_conn.commit()
        print("Data migration completed successfully")
    except Exception as e:
        print(f"An error occurred: {e}")
    finally:
        source_conn.close()
        target_conn.close()

# 替换以下变量为你的数据库凭证和表名
source_db = 'source_db'
target_db = 'target_db'
source_table = 'source_table'
target_table = 'target_table'

migrate_data_with_transformations(source_db, target_db, source_table, target_table)
```

### Gitea 迁移

首先找到配置文件 app.ini, 在其最后添加以下语句以允许局域网内迁移

```ini
[migrations]
ALLOW_LOCALNETWORKS = true
```

随后从源 web 页面上进入仓库列表，打开浏览器控制台，输入以下代码提取仓库名列表

```js
[...document.querySelectorAll("div.repo-title a.name")].map((i) => i.innerText);
```

之后将得到的仓库名列表复制到以下 python 脚本中的 repo_names 执行迁移, 注意设置正确的 repo_owner

```python
import requests

src_access_token = ''
tar_headers = {
    'Authorization': 'Bearer <tar_access_token>',
    'Content-Type': 'application/json'
}

def migrate_git_repo(repo_name: str, repo_owner: str):
    data = {
        "auth_token": src_access_token,
        "clone_addr": f"http://<gitea-src-ip>:3000/{repo_owner}/{repo_name}.git",
        "repo_name": repo_name,
        "repo_owner": repo_owner,
    }

    resp = requests.post('http://<gitea-tar-ip>:3000/api/v1/repos/migrate', json=data, headers=tar_headers)
    print(resp.text)

if __name__ == '__main__':
    repo_names = []
    for n in repo_names:
        migrate_git_repo(n, 'repo_owner')

```

### 移除 PDF 中的标注

首先，执行以下命令安装 pymupdf 包

```bash
pip install pymupdf -i https://pypi.tuna.tsinghua.edu.cn/simple
```

```python
import fitz  # 导入 PyMuPDF 库

# 打开 PDF 文件
doc = fitz.open("input.pdf")

# 遍历每一页
for page in doc:
    # 获取页面上的所有注释
    annots = page.annots()
    # 遍历所有注释并删除
    for annot in annots:
      annot_type = annot.type[1]  # 类型名称
      # 获取标注的内容
      annot_content = annot.info["content"] if "content" in annot.info else "No content"
      # 获取标注的矩形区域
      annot_rect = annot.rect
      # 打印标注信息
      print(f"  Type: {annot_type}, Content: '{annot_content}', Position: {annot_rect}")
      page.delete_annot(annot)

# 保存修改后的 PDF 文件
doc.save("output.pdf")
doc.close()
```
