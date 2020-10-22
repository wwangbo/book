# sync.Map源码分析

map是最常用的数据结构之一，但是go内置的map是并发不安全的，在go 1.9版本中，推出并发安全的map，即sync.Map。

数据结构:

```text
type Map struct {
  mu Mutex
  read atomic.Value // readOnly
  dirty map[interface{}]*entry
  misses int
}
type entry struct {
  p unsafe.Pointer // *interface{}
}
type readOnly struct {
  m       map[interface{}]*entry
  amended bool // true if the dirty map contains some key not in m.
}
```

```text
func (m *Map) Delete(key interface{}) {
  read, _ := m.read.Load().(readOnly)
  e, ok := read.m[key]
  if !ok && read.amended {//如果read里面没有数据，并且dirty里面有，那么应该去dirty里面删除数据
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)//double check确保在加锁之后没有被修改过
    e, ok = read.m[key]
    if !ok && read.amended {
      delete(m.dirty, key)//删除dirty里的数据
    }
    m.mu.Unlock()
  }
  if ok {//如果存在，则需要操作entry来删除
    e.delete()
  }
}
func (e *entry) delete() (hadValue bool) {
  for {
    p := atomic.LoadPointer(&e.p)//获取entry的数据
    if p == nil || p == expunged {//如果被删掉了或者被标记清除了，不管了
      return false
    }
    if atomic.CompareAndSwapPointer(&e.p, p, nil) {//通过cas去标记删除
      return true
    }
  }
}
```

```text
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
  read, _ := m.read.Load().(readOnly)//获取read map
  if e, ok := read.m[key]; ok && e.tryStore(&value) {//如果readmap里面有了，使用cas进行存储
    return
  }
​
  m.mu.Lock()//
  read, _ = m.read.Load().(readOnly)
  if e, ok := read.m[key]; ok {
    if e.unexpungeLocked() {
      // 如果read里面有，但是被标记删除了，则直接把value存到dirty里面
      m.dirty[key] = e
    }
    e.storeLocked(&value)
  } else if e, ok := m.dirty[key]; ok {//如果read没有。dirty有，那么更新dirty
    e.storeLocked(&value)
  } else {
    if !read.amended {
      // 如果都没有，那么将read的内容复制到dirty，但是不会复制已经标记删除的
      m.dirtyLocked()
      m.read.Store(readOnly{m: read.m, amended: true})
    }
    m.dirty[key] = newEntry(value)
  }
  m.mu.Unlock()
}
​
//expunged是一个任意指针，用于标记已从dirty中删除的
var expunged = unsafe.Pointer(new(interface{}))
​
func (e *entry) tryStore(i *interface{}) bool {
  for {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
      return false
    }
    if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
      return true
    }
  }
}
```

```text
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
  read, _ := m.read.Load().(readOnly)
  e, ok := read.m[key]//先去read map中获取
  if !ok && read.amended {//如果不存在并且read和dirty数据不一致
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)//double确保加锁之后不会被修改
    e, ok = read.m[key]
    if !ok && read.amended {//read不存在，从dirty中获取，然后存到read里面
      e, ok = m.dirty[key]
      m.missLocked()
    }
    m.mu.Unlock()
  }
  if !ok {//如果read里面没有，并且与dirty数据一致，则代表没有数据
    return nil, false
  }
  return e.load()//如果有的话，则直接在read里面获取数据
}
func (m *Map) missLocked() {
    m.misses++   
    if m.misses < len(m.dirty) {
      return
    }   
    // 如果 misses 小等 dirty 长度
    // 使用 dirty 覆盖 read，并清空 dirty
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
func (e *entry) load() (value interface{}, ok bool) {
  p := atomic.LoadPointer(&e.p)
  if p == nil || p == expunged {
    return nil, false
  }
  return *(*interface{})(p), true
}
```

* 如果read里面没有，且与dirty不一致，去dirty里面获取，dirty里面，则将dirty里面的存到read里面并清空dirty
* 将dirty里面的存到read里面并清空dirty的前提是misses的数量比dirty的长度小（也就是说dirty里面有数据）
* 如果read里面有，当然，直接从read里面拿（当然，要标记没被删除）

在获取数据时

* 如果read里面有，则使用cas进行存储（更新）
* 如果read有，但是被标记删除了，则存到dirty里面
* 如果read没有，dirty有，更新dirty
* 如果都没有，将read中没有标记删除的数据复制到dirty里面

在新增修改数据时

* 先查read有没有并且与dirty是否一致，如果没有且不一致，则dirty执行删除，如果没有且一致，则不需要进行任何操作，如果有，则直接在read和dirty中删除
* 但是，此时 “标记清除” 只是将 key 所对应的 entry 标记为了 “expunged”，key还在。什么时候 key 会被删除呢？需要经历一次 “`read -> dirty 同步` + `dirty -> read 同步`”。

在删数据时

可以看到，sync的设计非常精妙，虽然使用了两个map，但是所有的数据只存一份（在dirty有但是read没有，会在read中有标记，确保数据可以读到），在read有但是在dirty没有会在新增数据并且read和dirty里面都没有的时候全复制到dirty里（当然，标记了删除的就删掉了）。

### 总结

### 读数据

### 写数据

### 删数据

其中readOnly中map用来存放实际数据，amended=true表示，dirty里面有read中不存在的数据，那么如果从read里面读不到，就要去dirty里面读。使用这种方式进行存储的好处是，实际的值不需要在 read 和 dirty 中存储两次

read实际上是用的是readOnly类型，而dirty实际上是一个map\[interface{}\]\*entry类型。

* 新增数据，存到dirty
* 读数据，先读read，如果没有，再读dirty，刷新到read中

可以看到，sync.Map中使用了两个原生的map，一个read（readOnly），一个dirty。

