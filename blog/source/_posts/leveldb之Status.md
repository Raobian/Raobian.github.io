---
title: leveldb之Status
date: 2018-06-25 18:09:32
tags: leveldb
categories: leveldb
---
Status类是函数执行返回状态类，状态主要是用迭代类型enum Code类,
```
enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
  };
```
枚举code 长度为4

#### 私有变量state_, 
- state_[0..3] 为消息长度 
- state_[4] 为 返回码
- state_[5..] 为消息实体 
```
// OK status has a null state_.  Otherwise, state_ is a new[] array
// of the following form:
//    state_[0..3] == length of message
//    state_[4]    == code
//    state_[5..]  == message
const char* state_;
```
<!-- more -->
#### 构造声明与实现
include/leveldb/Status.h
```
class LEVELDB_EXPORT Status {
 public:
  // 构造 析构 
  // Create a success status.
  Status() noexcept : state_(nullptr) { }
  ~Status() { delete[] state_; }

  // 深拷贝构造 重载=操作符
  Status(const Status& rhs);
  Status& operator=(const Status& rhs);

  // 浅拷贝构造 重载=操作符
  Status(Status&& rhs) noexcept : state_(rhs.state_) { rhs.state_ = nullptr; }
  Status& operator=(Status&& rhs) noexcept;
```
实现
```
// 深拷贝需要重新分配资源
inline Status::Status(const Status& rhs) {
  state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_);
}

// 重新分配资源
inline Status& Status::operator=(const Status& rhs) {
  // The following condition catches both aliasing (when this == &rhs),
  // and the common case where both rhs and *this are ok.
  if (state_ != rhs.state_) {
    delete[] state_;
    // 状态不同，并且状态不为nullptr时调用 CopyState
    state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_);
  }
  return *this;
}

// 直接替换
inline Status& Status::operator=(Status&& rhs) noexcept {
  std::swap(state_, rhs.state_);
  return *this;
}
```
CopyState 函数实现在util/Status.cc
```
const char* Status::CopyState(const char* state) {
  uint32_t size;
  memcpy(&size, state, sizeof(size));   // 前4字节为大小
  char* result = new char[size + 5];    // 申请空间要加 5字节(长度与code)
  memcpy(result, state, size + 5);
  return result;
}
```

#### 几种状态

```
  // Return a success status.
  static Status OK() { return Status(); }

  // Return error status of an appropriate type.
  static Status NotFound(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotFound, msg, msg2);
  }
  static Status Corruption(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kCorruption, msg, msg2);
  }
  static Status NotSupported(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotSupported, msg, msg2);
  }
  static Status InvalidArgument(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kInvalidArgument, msg, msg2);
  }
  static Status IOError(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kIOError, msg, msg2);
  }
```
状态判断

```
  // ok状态 state_ == nullptr
  // Returns true iff the status indicates success.
  bool ok() const { return (state_ == nullptr); }

  // Returns true iff the status indicates a NotFound error.
  bool IsNotFound() const { return code() == kNotFound; }

  // Returns true iff the status indicates a Corruption error.
  bool IsCorruption() const { return code() == kCorruption; }

  // Returns true iff the status indicates an IOError.
  bool IsIOError() const { return code() == kIOError; }

  // Returns true iff the status indicates a NotSupportedError.
  bool IsNotSupportedError() const { return code() == kNotSupported; }

  // Returns true iff the status indicates an InvalidArgument.
  bool IsInvalidArgument() const { return code() == kInvalidArgument; }
```

#### 其他函数
```
  // Return a string representation of this status suitable for printing.
  // Returns the string "OK" for success.
  std::string ToString() const;

  // 第4字节为code
  Code code() const {
    return (state_ == nullptr) ? kOk : static_cast<Code>(state_[4]);
  }

  Status(Code code, const Slice& msg, const Slice& msg2);
  static const char* CopyState(const char* s);
};
```
实现

```
//构造 过程 计算长度 申请空间 赋值
Status::Status(Code code, const Slice& msg, const Slice& msg2) {
  assert(code != kOk);                  // ok状态 status == nullptr
  const uint32_t len1 = msg.size();
  const uint32_t len2 = msg2.size();
  const uint32_t size = len1 + (len2 ? (2 + len2) : 0);
  char* result = new char[size + 5];
  memcpy(result, &size, sizeof(size));
  result[4] = static_cast<char>(code);
  memcpy(result + 5, msg.data(), len1);
  if (len2) {
    result[5 + len1] = ':';
    result[6 + len1] = ' ';
    memcpy(result + 7 + len1, msg2.data(), len2);
  }
  state_ = result;
}

// C++ string 转化
std::string Status::ToString() const {
  if (state_ == nullptr) {
    return "OK";
  } else {
    char tmp[30];
    const char* type;
    switch (code()) {
      case kOk:
        type = "OK";
        break;
      case kNotFound:
        type = "NotFound: ";
        break;
      case kCorruption:
        type = "Corruption: ";
        break;
      case kNotSupported:
        type = "Not implemented: ";
        break;
      case kInvalidArgument:
        type = "Invalid argument: ";
        break;
      case kIOError:
        type = "IO error: ";
        break;
      default:
        snprintf(tmp, sizeof(tmp), "Unknown code(%d): ",
                 static_cast<int>(code()));
        type = tmp;
        break;
    }
    std::string result(type);
    uint32_t length;
    memcpy(&length, state_, sizeof(length));
    result.append(state_ + 5, length);
    return result;
  }
}
```

#### 简单使用
1.
```
Status s;
s = descriptor_log_->AddRecord(record);
if (s.ok()) {
    ...
```
2.
```
Status s = Status::Corruption("corrupted key for ", user_key);
Status::Corruption("log record too small");  
Status s = Status::IOError("Deleting DB during memtable compaction");  
Status::OK() ;  
s.ToString() ;
```