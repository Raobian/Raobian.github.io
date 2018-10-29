---
title: leveldb之key&Comparator
date: 2018-07-15 16:10:42
tags: leveldb
categories: leveldb
---
leveldb中存在多种key  
InternalKey & LookupKey  
![key](/images/keycomparator/key.jpg)  
skip_list_key  
![skip_list_key](/images/keycomparator/skip_list_key.jpg)
<!-- more -->
## key
#### ValueType
```
enum ValueType {
  kTypeDeletion = 0x0,  //标记删除的
  kTypeValue = 0x1
};
```
#### SequenceNumber
```
typedef uint64_t SequenceNumber;
// 低八位用于保存type
// We leave eight bits empty at the bottom so a type and sequence#
// can be packed together into 64-bits.
static const SequenceNumber kMaxSequenceNumber =
    ((0x1ull << 56) - 1);
```
#### ParsedInternalKey 解析的内部key，操作使用
| user_key | sequence | type |
```
struct ParsedInternalKey {
  Slice user_key;
  SequenceNumber sequence;
  ValueType type;

  ParsedInternalKey() { }  // Intentionally left uninitialized (for speed)
  ParsedInternalKey(const Slice& u, const SequenceNumber& seq, ValueType t)
      : user_key(u), sequence(seq), type(t) { }
  std::string DebugString() const;
};
```
转换函数InternalKey
```
inline bool ParseInternalKey(const Slice& internal_key,
                             ParsedInternalKey* result) {
  const size_t n = internal_key.size();
  if (n < 8) return false;
  uint64_t num = DecodeFixed64(internal_key.data() + n - 8); // seq type
  unsigned char c = num & 0xff;
  result->sequence = num >> 8;
  result->type = static_cast<ValueType>(c);
  result->user_key = Slice(internal_key.data(), n - 8);
  return (c <= static_cast<unsigned char>(kTypeValue));
}
```
#### InternalKey 内部key 储存使用
| user_key | sequence | type |
```
// Modules in this directory should keep internal keys wrapped inside
// the following class instead of plain strings so that we do not
// incorrectly use string comparisons instead of an InternalKeyComparator.
class InternalKey {
 private:
  std::string rep_;
 public:
  InternalKey() { }   // Leave rep_ as empty to indicate it is invalid
  InternalKey(const Slice& user_key, SequenceNumber s, ValueType t) {
    // AppendInternalKey
    AppendInternalKey(&rep_, ParsedInternalKey(user_key, s, t));
  }
  void DecodeFrom(const Slice& s) { rep_.assign(s.data(), s.size()); } // assign为string替换方法
  Slice Encode() const {
    assert(!rep_.empty());
    return rep_;
  }

  Slice user_key() const { return ExtractUserKey(rep_); }

  void SetFrom(const ParsedInternalKey& p) {
    rep_.clear();
    AppendInternalKey(&rep_, p);
  }

  void Clear() { rep_.clear(); }

  std::string DebugString() const;
}
```
AppendInternalKey
```
void AppendInternalKey(std::string* result, const ParsedInternalKey& key) {
  result->append(key.user_key.data(), key.user_key.size()); // user key
  PutFixed64(result, PackSequenceAndType(key.sequence, key.type)); // seq type
}
```
#### LookupKey  MemTable使用
| size | InternalKey |
```
// A helper class useful for DBImpl::Get()
class LookupKey {
 public:
  // Initialize *this for looking up user_key at a snapshot with
  // the specified sequence number.
  LookupKey(const Slice& user_key, SequenceNumber sequence);

  ~LookupKey();

  // Return a key suitable for lookup in a MemTable.
  Slice memtable_key() const { return Slice(start_, end_ - start_); }

  // Return an internal key (suitable for passing to an internal iterator)
  Slice internal_key() const { return Slice(kstart_, end_ - kstart_); }

  // Return the user key
  Slice user_key() const { return Slice(kstart_, end_ - kstart_ - 8); }

 private:
  // We construct a char array of the form:
  //    klength  varint32               <-- start_
  //    userkey  char[klength]          <-- kstart_
  //    tag      uint64
  //                                    <-- end_
  // The array is a suitable MemTable key.
  // The suffix starting with "userkey" can be used as an InternalKey.
  const char* start_;
  const char* kstart_;
  const char* end_;
  char space_[200];      // Avoid allocation for short keys

  // No copying allowed
  LookupKey(const LookupKey&);
  void operator=(const LookupKey&);
};
```
#### SkipList 中的key
| key size | user_key | seq | type | value size | value |  
同时存储了key和value

## Comparator
Comparator为抽象类，默认比较函数为字节比较

```
// A Comparator object provides a total order across slices that are
// used as keys in an sstable or a database.  A Comparator implementation
// must be thread-safe since leveldb may invoke its methods concurrently
// from multiple threads.
// 实现类必须是多线程安全的
class LEVELDB_EXPORT Comparator {
 public:
  virtual ~Comparator();

  // Three-way comparison.  Returns value:
  //   < 0 iff "a" < "b",
  //   == 0 iff "a" == "b",
  //   > 0 iff "a" > "b"
  // 比较函数
  virtual int Compare(const Slice& a, const Slice& b) const = 0;

  // The name of the comparator.  Used to check for comparator
  // mismatches (i.e., a DB created with one comparator is
  // accessed using a different comparator.
  //
  // The client of this package should switch to a new name whenever
  // the comparator implementation changes in a way that will cause
  // the relative ordering of any two keys to change.
  //
  // Names starting with "leveldb." are reserved and should not be used
  // by any clients of this package.
  // 用名字区分comparator
  // 以'leveldb.'开头被保留，不能被clients使用
  virtual const char* Name() const = 0;

  // Advanced functions: these are used to reduce the space requirements
  // for internal data structures like index blocks.

  // If *start < limit, changes *start to a short string in [start,limit).
  // Simple comparator implementations may return with *start unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  // 两个字符串间的最短字符串
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const = 0;

  // Changes *key to a short string >= *key.
  // Simple comparator implementations may return with *key unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  // 最短的>= *key的字符串
  virtual void FindShortSuccessor(std::string* key) const = 0;
};

// Return a builtin comparator that uses lexicographic byte-wise
// ordering.  The result remains the property of this module and
// must not be deleted.
LEVELDB_EXPORT const Comparator* BytewiseComparator();
```

#### 实现类 BytewiseComparatorImpl
```
class BytewiseComparatorImpl : public Comparator {
 public:
  BytewiseComparatorImpl() { }

  virtual const char* Name() const {
    // 命名为 leveldb.BytewiseComparator
    return "leveldb.BytewiseComparator";
  }

  // 字符串比较直接调用slice的compare方法
  virtual int Compare(const Slice& a, const Slice& b) const {
    return a.compare(b);
  }

  // 找到两个字符串间的最短字符串 
  // 如start="helloword"  limit="hellwzbdr" 结果将start 改为"hellp"
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const {
    // Find length of common prefix
    // 首先计算共同前缀字符串的长度
    size_t min_length = std::min(start->size(), limit.size());
    size_t diff_index = 0;
    while ((diff_index < min_length) &&
           ((*start)[diff_index] == limit[diff_index])) {
      diff_index++;
    }
    // 找到不同index  index=4

    if (diff_index >= min_length) {
      // Do not shorten if one string is a prefix of the other
      // 一个字符串是另一个字符串的前缀，不操作
    } else {
      uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
      // start不同位 < 0xff, start不同+1位 < limit不同位
      // 如o < 0xff, w < z
      if (diff_byte < static_cast<uint8_t>(0xff) &&
          diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) {
        (*start)[diff_index]++;
        start->resize(diff_index + 1);
        // start 改为 "hellp"
        assert(Compare(*start, limit) < 0);
      }
    }
  }

  virtual void FindShortSuccessor(std::string* key) const {
    // Find first character that can be incremented
    // 找到第一个可以++的字符，执行++后，截断字符串；  
    // 如果找不到说明*key的字符都是0xff啊，那就不作修改，直接返回
    size_t n = key->size();
    for (size_t i = 0; i < n; i++) {
      const uint8_t byte = (*key)[i];
      if (byte != static_cast<uint8_t>(0xff)) {
        (*key)[i] = byte + 1;
        key->resize(i+1);
        return;
      }
    }
    // *key is a run of 0xffs.  Leave it alone.
  }
};
}  // namespace

static port::OnceType once = LEVELDB_ONCE_INIT;
static const Comparator* bytewise;

static void InitModule() {
  bytewise = new BytewiseComparatorImpl;
}

const Comparator* BytewiseComparator() {
  port::InitOnce(&once, InitModule);
  return bytewise;
}
```

#### 实现类 InternalKeyComparator

```
// A comparator for internal keys that uses a specified comparator for
// the user key portion and breaks ties by decreasing sequence number.
class InternalKeyComparator : public Comparator {
 private:
  const Comparator* user_comparator_;  // user key的比较函数
 public:
  explicit InternalKeyComparator(const Comparator* c) : user_comparator_(c) { }
  virtual const char* Name() const;
  virtual int Compare(const Slice& a, const Slice& b) const;
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const;
  virtual void FindShortSuccessor(std::string* key) const;

  const Comparator* user_comparator() const { return user_comparator_; }

  int Compare(const InternalKey& a, const InternalKey& b) const;
};

```
###### Name
```
const char* InternalKeyComparator::Name() const {
  return "leveldb.InternalKeyComparator";
}
```
###### Compare
```
// 先使用user_compair比较user key，不相等就直接返回，
// 相等再比较sequence number | value type, 降序
// 这里，没有比较后面的value了，因为sequence number是唯一的
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```
###### FindShortestSeparator & FindShortSuccessor
```
void InternalKeyComparator::FindShortestSeparator(
      std::string* start,
      const Slice& limit) const {
  // user key
  // Attempt to shorten the user portion of the key
  Slice user_start = ExtractUserKey(*start);
  Slice user_limit = ExtractUserKey(limit);
  std::string tmp(user_start.data(), user_start.size());
  user_comparator_->FindShortestSeparator(&tmp, user_limit);
  if (tmp.size() < user_start.size() &&
      user_comparator_->Compare(user_start, tmp) < 0) {
    // 使用最大的sequence number以保证是最新的*start
    // User key has become shorter physically, but larger logically.
    // Tack on the earliest possible number to the shortened user key.
    PutFixed64(&tmp, PackSequenceAndType(kMaxSequenceNumber,kValueTypeForSeek));
    assert(this->Compare(*start, tmp) < 0);
    assert(this->Compare(tmp, limit) < 0);
    start->swap(tmp);
  }
}

// 取出internal key的user key字段，根据internal key字段找到并替换key,如果key被替换了，
// 就用新的key更新Internal Key，并使用最大的sequence number。否则保持不变。
void InternalKeyComparator::FindShortSuccessor(std::string* key) const {
  Slice user_key = ExtractUserKey(*key);
  std::string tmp(user_key.data(), user_key.size());
  user_comparator_->FindShortSuccessor(&tmp);
  if (tmp.size() < user_key.size() &&
      user_comparator_->Compare(user_key, tmp) < 0) {
    // User key has become shorter physically, but larger logically.
    // Tack on the earliest possible number to the shortened user key.
    PutFixed64(&tmp, PackSequenceAndType(kMaxSequenceNumber,kValueTypeForSeek));
    assert(this->Compare(*key, tmp) < 0);
    key->swap(tmp);
  }
}
```
