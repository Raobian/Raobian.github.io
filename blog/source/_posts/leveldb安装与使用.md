---
title: leveldb安装与使用
date: 2018-06-22 15:09:18
tags: leveldb
categories: leveldb
---
**leveldb** Google C++ kv数据库。代码风格优秀。
涉及到了skip list、内存KV table、LRU cache管理、table文件存储、operation log系统等。

## 安装
### 下载

```
git clone https://github.com/google/leveldb.git
```
### 编译 安装

```
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
make && make install
```
<!-- more -->
## 使用
hello_leveldb.cc

```
#include <iostream>
#include <cassert>
#include <cstdlib>
#include <string>
#include <leveldb/db.h>

using namespace std;

int main(void)
{
    leveldb::DB *db = nullptr;
    leveldb::Options options;
    
    // 如果数据库不存在就创建
    options.create_if_missing = true;
    
    // 创建的数据库 /tmp/testdb
    leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
    assert(status.ok());
    
    string key = "akey";
    string value = "avalue";
    string gvalue;
    
    // Put
    leveldb::Status s = db->Put(leveldb::WriteOptions(), key, value);
    if (!s.ok()) {
        // 写入失败
        cout << s.ToString() << endl;
        return 0;
    }

    // Get
    s = db->Get(leveldb::ReadOptions(), key, &gvalue);
    cout << (s.ok() ? gvalue : s.ToString()) << endl;
    
    delete db;
    return 0;
}
```
### 编译

```
g++ hello_leveldb.cc -o test -lpthread -lleveldb
```
### 运行结果

```
# ./test
avalue
```

只做了简单的put get操作，还有delete、snapshot等操作。 