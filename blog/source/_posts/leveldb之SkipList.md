---
title: leveldb之SkipList
date: 2018-06-26 17:02:18
tags: leveldb
categories: leveldb
---
![skiplist](/images/skiplist/skiplist.gif)
![skiplist](/images/skiplist/skiplist.jpg)
Memtable是leveldb的核心之一，而Memtable使用跳表SkipList实现。 

skiplist的效率可以和平衡树媲美，平均O(logN)，最坏O(N)的查询效率，但是用skiplist实现比平衡树实现简单，所以很多程序用跳跃链表来代替平衡树。
<!-- more -->

## 内存屏障
leveldb是支持多线程操作的，但是skiplist并没有使用linux下锁，信号量来实现同步控制，据说是因为锁机制导致某个线程占有资源，其他线程阻塞的情况，导致系统资源利用率降低。所以leveldb采用的是内存屏障来实现同步机制。

内存屏障参考之前文章《leveldb之AtomicPointer》

## SkipList
先看节点SkipList::Node
### SkipList::Node
```
// Implementation details follow
template<typename Key, class Comparator>
struct SkipList<Key,Comparator>::Node {

  // C++中的explicit关键字只能用于修饰只有一个参数的类构造函数, 它的作用是表明该构造函数是显示的, 
  //而非隐式的, 跟它相对应的另一个关键字是implicit, 意思是隐藏的,类构造函数默认情况下即声明为implicit(隐式).
  explicit Node(const Key& k) : key(k) { }

  Key const key;

  // 封装next[n]的get set方法，内存屏障
  // Accessors/mutators for links.  Wrapped in methods so we can
  // add the appropriate barriers as necessary.
  Node* Next(int n) {
    assert(n >= 0);
    // Use an 'acquire load' so that we observe a fully initialized
    // version of the returned Node.
    return reinterpret_cast<Node*>(next_[n].Acquire_Load());
  }
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.
    next_[n].Release_Store(x);
  }

  // 无内存屏障set get next[n]
  // No-barrier variants that can be safely used in a few locations.
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return reinterpret_cast<Node*>(next_[n].NoBarrier_Load());
  }
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].NoBarrier_Store(x);
  }

 private:
  // next数组，大小为节点高度，初始只定义0层，其余根据高度来动态申请空间
  // Array of length equal to the node height.  next_[0] is lowest level link.
  port::AtomicPointer next_[1];
};
```

### SkipList
#### 私有变量
```
private:
  struct Node;
  
private:
  enum { kMaxHeight = 12 };         // 定义最大高度

  // 比较类 外部传入
  // Immutable after construction
  Comparator const compare_;
  // arena 外部传入
  Arena* const arena_;    // Arena used for allocations of nodes

  // 头结点
  Node* const head_;

  // 当前最大高度，只被Insert修改
  // Modified only by Insert().  Read racily by readers, but stale
  // values are ok.
  port::AtomicPointer max_height_;   // Height of the entire list
  
  // 随机数类，插入时 可用于获取一个随机高度
  // Read/written only by Insert().
  Random rnd_;
```
#### 构造析构
##### 定义
```
  // Create a new SkipList object that will use "cmp" for comparing keys,
  // and will allocate memory using "*arena".  Objects allocated in the arena
  // must remain allocated for the lifetime of the skiplist object.
  explicit SkipList(Comparator cmp, Arena* arena);
  
  // No copying allowed
  SkipList(const SkipList&);
  void operator=(const SkipList&);
```
##### 实现
```
template<typename Key, class Comparator>
SkipList<Key,Comparator>::SkipList(Comparator cmp, Arena* arena)
    : compare_(cmp),
      arena_(arena),
      head_(NewNode(0 /* any key will do */, kMaxHeight)),
      max_height_(reinterpret_cast<void*>(1)),
      rnd_(0xdeadbeef) {
  for (int i = 0; i < kMaxHeight; i++) {
    head_->SetNext(i, nullptr);
  }
}  
```
compare arena 外部传入，  
头结点为新建key为0，高度为12的Node  
list最大高度初始化为1  
rnd_ 0xdeadbeef  
头结点的所有next都指向nullptr
```
//  新建Node
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node*
SkipList<Key,Comparator>::NewNode(const Key& key, int height) {
  // 根据 高度 申请空间
  char* mem = arena_->AllocateAligned(
      sizeof(Node) + sizeof(port::AtomicPointer) * (height - 1));
  return new (mem) Node(key);
}
```
new (mem) Node(key)为placement new操作，空间申请和销毁需要手动操作


#### 私有方法
```
  // 获取最大高度
  inline int GetMaxHeight() const {
    return static_cast<int>(
        reinterpret_cast<intptr_t>(max_height_.NoBarrier_Load()));
  }
  
  // 获取随机高度
  int RandomHeight();
  // key 是否相等
  bool Equal(const Key& a, const Key& b) const { return (compare_(a, b) == 0); }

  // key是否在某结点之后
  // Return true if key is greater than the data stored in "n"
  bool KeyIsAfterNode(const Key& key, Node* n) const;

  // 查找每层key的前节点，填充到prev
  // Return the earliest node that comes at or after key.
  // Return nullptr if there is no such node.
  //
  // If prev is non-null, fills prev[level] with pointer to previous
  // node at "level" for every level in [0..max_height_-1].
  Node* FindGreaterOrEqual(const Key& key, Node** prev) const;

  //  找到一个小于key的节点
  // Return the latest node with a key < key.
  // Return head_ if there is no such node.
  Node* FindLessThan(const Key& key) const;

  // 最后一个节点
  // Return the last node in the list.
  // Return head_ if list is empty.
  Node* FindLast() const;
```
##### 1. 产生随机高度，每层1/4概率抽样
```
template<typename Key, class Comparator>
int SkipList<Key,Comparator>::RandomHeight() {
  // 1/4概率抽样
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```
##### 2. key是否在某结点之后，对比key大小
```
template<typename Key, class Comparator>
bool SkipList<Key,Comparator>::KeyIsAfterNode(const Key& key, Node* n) const {
  // null n is considered infinite
  return (n != nullptr) && (compare_(n->key, key) < 0);
}
```
##### 3. 查找每层key的前节点，填充到prev的每一层，返回大于等于key的节点
```
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node* SkipList<Key,Comparator>::FindGreaterOrEqual(const Key& key, Node** prev)
    const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {        
      // Keep searching in this list
      x = next;
    } else {                                // x <= key < next
      if (prev != nullptr) prev[level] = x; // 填充
      if (level == 0) {                     // 查找结束 返回0层下一个节点
        return next;
      } else {                              // 查找下一层
        // Switch to next list
        level--;
      }
    }
  }
}
```
##### 4. 找到key的前一个节点
```
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node*
SkipList<Key,Comparator>::FindLessThan(const Key& key) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    assert(x == head_ || compare_(x->key, key) < 0);
    Node* next = x->Next(level);
    if (next == nullptr || compare_(next->key, key) >= 0) { // next >= key > x
      if (level == 0) {
        return x;
      } else {
        // Switch to next list
        level--;
      }
    } else {
      x = next;
    }
  }
}
```
##### 5. 找到最后一个节点
```
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node* SkipList<Key,Comparator>::FindLast()
    const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (next == nullptr) {
      if (level == 0) {
        return x;
      } else {
        // Switch to next list
        level--;
      }
    } else {
      x = next;
    }
  }
}
```
#### 公共方法
```
 public:
  // Insert key into the list.
  // REQUIRES: nothing that compares equal to key is currently in the list.
  void Insert(const Key& key);

  // Returns true iff an entry that compares equal to key is in the list.
  bool Contains(const Key& key) const;
};
```
##### 1. 插入
主要是查找前置节点，并插入到前置节点后边
```
template<typename Key, class Comparator>
void SkipList<Key,Comparator>::Insert(const Key& key) {
  // TODO(opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
  Node* prev[kMaxHeight];
  
  // 查找到前置节点到prev，下一个节点为x
  Node* x = FindGreaterOrEqual(key, prev);

  // 禁止重复插入
  // Our data structure does not allow duplicate insertion
  assert(x == nullptr || !Equal(key, x->key));

  // 获取随机高度，如果超过当前最大高度，填充prev超出部分为head，更新当前最大高度
  int height = RandomHeight();
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    //fprintf(stderr, "Change height from %d to %d\n", max_height_, height);

    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (nullptr), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since nullptr sorts after all
    // keys.  In the latter case the reader will use the new node.
    max_height_.NoBarrier_Store(reinterpret_cast<void*>(height));
  }

  // 新建节点，x->next = prev->next, prev->next = x
  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}
```
##### 2. 包含
找到大于等于key的节点，判断key是否相等
```
template<typename Key, class Comparator>
bool SkipList<Key,Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```

#### 迭代器Iterator
内部类:内部类就是外部类的友元类。注意友元类的定义，内部类可以通过外部类的对象参数来访问外部类中的所有成员。但是外部类不是内部类的友元。  
这个内部类是一个独立的类，它不属于外部类，更不能通过外部类的对象去调用内部类。外部类对内部类没有任何优越的访问权限。
```
  // Iteration over the contents of a skip list
  class Iterator {
   public:
    // 基于list，初始状态为不可用
    // Initialize an iterator over the specified list.
    // The returned iterator is not valid.
    explicit Iterator(const SkipList* list);

    // 是否可用
    // Returns true iff the iterator is positioned at a valid node.
    bool Valid() const;

    // 返回当前位置的key，需要可用状态
    // Returns the key at the current position.
    // REQUIRES: Valid()
    const Key& key() const;

    // 跳到下一个位置
    // Advances to the next position.
    // REQUIRES: Valid()
    void Next();
    
    // 跳到上一个位置
    // Advances to the previous position.
    // REQUIRES: Valid()
    void Prev();

    // 跳到目标位置
    // Advance to the first entry with a key >= target
    void Seek(const Key& target);

    // 跳到首位
    // Position at the first entry in list.
    // Final state of iterator is Valid() iff list is not empty.
    void SeekToFirst();

    // 跳到末位
    // Position at the last entry in list.
    // Final state of iterator is Valid() iff list is not empty.
    void SeekToLast();

   private:
    const SkipList* list_;          // 要迭代的跳表
    Node* node_;                    // 当前节点
    // Intentionally copyable
  };
```
##### 构造
```
template<typename Key, class Comparator>
inline SkipList<Key,Comparator>::Iterator::Iterator(const SkipList* list) {
  list_ = list;
  node_ = nullptr;
}
```
##### 方法
```
// 通过当前节点是否有效判断是否可用
template<typename Key, class Comparator>
inline bool SkipList<Key,Comparator>::Iterator::Valid() const {
  return node_ != nullptr;
}

// 返回当前节点key
template<typename Key, class Comparator>
inline const Key& SkipList<Key,Comparator>::Iterator::key() const {
  assert(Valid());
  return node_->key;
}

// 当前节点跳到0层next
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Next() {
  assert(Valid());
  node_ = node_->Next(0);
}

// 当前节点跳到prev
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Prev() {
  // Instead of using explicit "prev" links, we just search for the
  // last node that falls before key.
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
}

// 查找大于等于key的下一个节点
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Seek(const Key& target) {
  node_ = list_->FindGreaterOrEqual(target, nullptr);
}

// 跳到head的0层next
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::SeekToFirst() {
  node_ = list_->head_->Next(0);
}

// 找到最后节点，跳转
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::SeekToLast() {
  node_ = list_->FindLast();
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
}
```