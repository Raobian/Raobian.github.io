---
title: leveldb之AtomicPointer
date: 2018-06-26 14:27:17
tags: leveldb
categories: leveldb
---
AtomicPointer 原子指针，并不是原子操作指针，因为指针赋值本身就是原子操作。而leveldb中的原子指针，涉及的是无锁编程，无锁编程涉及到内存屏障。
#### 内存屏障
- 硬件层的内存屏障分为两种：Load Barrier 和 Store Barrier即读屏障和写屏障。
- 内存屏障有两个作用：
```
1. 阻止屏障两侧的指令重排序；
2. 强制把写缓冲区/高速缓存中的脏数据等写回主内存，让缓存中相应的数据失效。
```
- 对于Load Barrier来说，在指令前插入Load Barrier，可以让高速缓存中的数据失效，强制从新从主内存加载数据；
- 对于Store Barrier来说，在指令后插入Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。

leveldb中定义内存屏障  
只贴了x86 linux，其他平台可以参考源码
```
// Gcc on x86
#elif defined(ARCH_CPU_X86_FAMILY) && defined(__GNUC__)
inline void MemoryBarrier() {
  // See http://gcc.gnu.org/ml/gcc/2003-04/msg01180.html for a discussion on
  // this idiom. Also see http://en.wikipedia.org/wiki/Memory_ordering.
  __asm__ __volatile__("" : : : "memory");
}
#define LEVELDB_HAVE_MEMORY_BARRIER
```
<!-- more -->
这里的volatile主要是用来防止编译器优化的。  
就是告诉编译器：不要因为性能优化而将这些代码重排，我需要清清爽爽的保持这三块代码块的顺序（代码块内部是否重排不是这里的volatile管辖范围了）。

如果你想实现下面这样的功能，那你可以考虑内存屏障：   
修改一个内存中的变量之后，其余的 CPU 和 Cache 里面该变量的原始数据失效，必须从内存中重新获取这个变量的值。 
这保证了这个变量对 CPU 和 Cache 是「可见的」，leveldb 就使用了这个特性

#### AtomicPointer
```
// AtomicPointer built using platform-specific MemoryBarrier().
#if defined(LEVELDB_HAVE_MEMORY_BARRIER)
class AtomicPointer {
 private:
  void* rep_;
 public:
  AtomicPointer() { }
  explicit AtomicPointer(void* p) : rep_(p) {}
  
  // 不使用内存屏障的读操作，即不同步的读操作
  inline void* NoBarrier_Load() const { return rep_; }
  // 同上，是不同步的写操作
  inline void NoBarrier_Store(void* v) { rep_ = v; }
  
  
  // 使用内存屏障的读操作，即同步读
  inline void* Acquire_Load() const {
    void* result = rep_;
    // 添加一个内存屏障，后面会有原理介绍
    MemoryBarrier();
    return result;
  }
  
  // 使用内存屏障的写操作，即同步写
  inline void Release_Store(void* v) {
    MemoryBarrier();
    rep_ = v;
  }
};
```
无锁编程的概念做一般应用层开发的会较少接触到，因为多线程的时候对共享资源的操作一般是用锁来完成的。  
锁本身对这个任务完成的很好，但是存在性能的问题，也就是在对性能要求很高的，高并发的场景下，锁会带来性能瓶颈。所以在一些如数据库这样的应用或者linux 内核里经常会看到一些无锁的并发编程。  
锁是一个高层次的接口，隐藏了很多并发编程时会出现的非常古怪的问题。当不用锁的时候，就要考虑这些问题。主要有两个方面的影响：编译器对指令的排序和cpu对指令的排序。他们排序的目的主要是优化和提高效率。排序的原则是在单核单线程下最终的效果不会发生改变。单核多线程的时候，编译器的乱序就会带来问题，多核的时候，又会涉及cpu对指令的乱序。memory-ordering-at-compile-time和memory-reordering-caught-in-the-act里提到了乱序导致的问题。  
注意到其中几个成员函数都是inline，如果不是inline，其实没有必要加上内存屏障，因为函数能够提供很强的内存屏障保证。

下面针对Acquire_Load和Release_Store假设一个场景：
```
//thread1：
Object.var1 = a;
Object.var2 = b;
Object.var2 = c;
atomicpointer.Release_Store(p);

//thread2
user_pointer = atomicpointer.Acquire_Load();
get Object.va1
get Object.var2
get Object.var3
```
结合之前的分析，可以很容易明白此时内存屏障保证了在线程1里指针赋值之前对象的所有操作都已经完成，而在线程2里面保证了取出指针后，才会开始获取新的对象内容。这符合程序的顺序逻辑。  
注意acquire，release模型适合单生产者和单消费者的模型，如果有多个生产者，那么现有的保障是不足的，会涉及到原子性的问题。