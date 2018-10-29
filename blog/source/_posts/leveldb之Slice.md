---
title: leveldb之Slice
date: 2018-06-25 16:46:43
tags: leveldb
categories: leveldb
---
LevelDB 中的字符串没有使用 std:string，而是使用Slice。题外话，redis没有使用C语言中的字符串，而是自己构建了SDS这样一种简单动态字符串。Slice

include/leveldb/Slice.h
<!-- more -->
```
class LEVELDB_EXPORT Slice {
 public:
  // 构造函数 创建空的slice
  // Create an empty slice.
  Slice() : data_(""), size_(0) { }

  // 构造函数 使用d[0, n - 1] 构造
  // Create a slice that refers to d[0,n-1].
  Slice(const char* d, size_t n) : data_(d), size_(n) { }

  // 构造函数 使用C++ string 构造
  // Create a slice that refers to the contents of "s"
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) { }

  // 构造函数 使用指定长度字符串指针 构造
  // Create a slice that refers to s[0,strlen(s)-1]
  Slice(const char* s) : data_(s), size_(strlen(s)) { }

  // 采用默认拷贝构造，直接赋值
  // Intentionally copyable.
  Slice(const Slice&) = default;
  Slice& operator=(const Slice&) = default;


  // ========== 常用函数 ===========

  // 返回字符串指针
  // Return a pointer to the beginning of the referenced data
  const char* data() const { return data_; }

  // 返回数据长度
  // Return the length (in bytes) of the referenced data
  size_t size() const { return size_; }

  // 是否为空
  // Return true iff the length of the referenced data is zero
  bool empty() const { return size_ == 0; }

  // 重载[]，通过slice[n]返回第n个字符
  // Return the ith byte in the referenced data.
  // REQUIRES: n < size()
  char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
  }
  
  // 清空
  // Change this slice to refer to an empty array
  void clear() { data_ = ""; size_ = 0; }

  // 去除前缀n个字符
  // Drop the first "n" bytes from this slice.
  void remove_prefix(size_t n) {
    assert(n <= size());
    data_ += n;
    size_ -= n;
  }

  // string 转化
  // Return a string that contains the copy of the referenced data.
  std::string ToString() const { return std::string(data_, size_); }

  // 比较
  // Three-way comparison.  Returns value:
  //   <  0 iff "*this" <  "b",
  //   == 0 iff "*this" == "b",
  //   >  0 iff "*this" >  "b"
  int compare(const Slice& b) const;

  // 是否以slice x为前缀
  // Return true iff "x" is a prefix of "*this"
  bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) &&
            (memcmp(data_, x.data_, x.size_) == 0));
  }

 private:
  const char* data_;        // 数据指针
  size_t size_;             // 数据长度
}
```

```
// 重载==操作符，用于判断两个Slice是否相等
inline bool operator==(const Slice& x, const Slice& y) {
  return ((x.size() == y.size()) &&
          (memcmp(x.data(), y.data(), x.size()) == 0));
}

// 重载!=操作符，用于判断两个Slice是否不等
inline bool operator!=(const Slice& x, const Slice& y) {
  return !(x == y);
}

// 比较函数实现
inline int Slice::compare(const Slice& b) const {
  const size_t min_len = (size_ < b.size_) ? size_ : b.size_;
  int r = memcmp(data_, b.data_, min_len);
  if (r == 0) {
    if (size_ < b.size_) r = -1;
    else if (size_ > b.size_) r = +1;
  }
  return r;
}
```

简单使用

```
//声明定义一个空字符串
Slice slice；
//字符串指针初始化Slice
Slice s1 = "hello";
const char* p="world"
Slice s2(p);
//获取Slice的字符串
s2.data();
//获取Slice字符串的长度
s2.size()
```
==注意：== 使用Slice时需要格外小心，因为Slice引用的外部数组是由Slice的使用者保证在Slice的生命周期内外部数组是有效的。
比如下面的代码中存在bug：

```
Slice slice; 
if (...) {
  std::string str = ...; 
  slice = str; 
} 
Use(slice); 
```
当if语句的作用域结束时，str会被析构，slice指向的外部空间就不存在了。

