---
title: leveldb之Arena
date: 2018-06-26 14:37:49
tags: leveldb  
categories: leveldb
---
Arena 内存池，避免频繁的new/delete，减少内存申请和释放的开销。  
模型如下
![arena](/images/arena/arena.jpg)
![arena_model](/images/arena/arena_model.jpeg)
<!-- more -->
##### 成员变量

```
  // Allocation state
  char* alloc_ptr_;                 // 内存偏移，指向未使用内存首地址
  size_t alloc_bytes_remaining_;    // 剩余已申请的内存

  // Array of new[] allocated memory blocks
  std::vector<char*> blocks_;       // 记录每个block的首地址

  // Total memory usage of the arena.
  port::AtomicPointer memory_usage_;    // 已使用内存大小
```

##### 构造析构
```
Arena::Arena() : memory_usage_(0) {
  alloc_ptr_ = nullptr;  // First allocation will allocate a block
  alloc_bytes_remaining_ = 0;
  // vector会调用默认构造函数初始化
}

Arena::~Arena() {
  for (size_t i = 0; i < blocks_.size(); i++) {
    delete[] blocks_[i];
  }
}
```
不允许拷贝构造和 =赋值
```
  // No copying allowed
  Arena(const Arena&);
  void operator=(const Arena&);
```

##### 申请内存
block大小
```
static const int kBlockSize = 4096;
```
声明
```
  // 申请内存
  // Return a pointer to a newly allocated memory block of "bytes" bytes.
  char* Allocate(size_t bytes);

  // 申请内存对齐
  // Allocate memory with the normal alignment guarantees provided by malloc
  char* AllocateAligned(size_t bytes);
  
  private:
  char* AllocateFallback(size_t bytes);
  char* AllocateNewBlock(size_t block_bytes);
```
实现  
过程中分三种情况：
1. 剩余内存足够，直接使用剩余内存分配
1. 剩余内存不足，且申请空间 > kBlockSize/4时，申请一个本次申请大小的block
1. 剩余内存不足，且申请空间 <= kBlockSize/4时，申请一个kBlockSize大小的block

```
// 分配内存
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);
  
  // 1.剩余内存足够，直接使用剩余内存分配
  if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  return AllocateFallback(bytes);
}

char* Arena::AllocateFallback(size_t bytes) {
  // 2.剩余内存不足，且申请空间 > kBlockSize/4时，申请一个本次申请大小的block
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // 3.剩余内存不足，且申请空间 <= kBlockSize/4时，申请一个kBlockSize大小的block
  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}

// 申请指定大小block
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);        // 添加到blocks_中
  // 更新MemoryUsage = 已申请内存 + 本次申请 + 一个指针大小
  memory_usage_.NoBarrier_Store(
      reinterpret_cast<void*>(MemoryUsage() + block_bytes + sizeof(char*)));
  return result;
}
```
###### 对齐申请
```
char* Arena::AllocateAligned(size_t bytes) {
  //用于判断对齐的大小，我64位电脑sizeof(void*)=8，不大于8，所以对齐大小为8。
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  
  // 对齐肯定为2的n次幂 如8==1000 & 7==0111 = 0
  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
  
  // alloc_ptr_ & (align-1) 相当于对align取模，如9==1001&7==0111 == 9%8 = 1
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);
  
  // 计算溢出 即额外需要多少补齐
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  
  // 计算申请大小 = 要申请大小 + 补齐字节
  size_t needed = bytes + slop;
  char* result;
  
  // 有剩余，直接对齐分配
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // 重申申请 总是对齐的
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0);
  return result;
}


```

###### 提供MemoryUsage用于查看内存池总容量
```
  // Returns an estimate of the total memory usage of data allocated
  // by the arena.
  size_t MemoryUsage() const {
    return reinterpret_cast<uintptr_t>(memory_usage_.NoBarrier_Load());
  }
```

