# List源码分析

### 源码分析 <a id="h"></a>

List实际上是由一个个Element组成，List和Element的定义如下:

```text
type List struct {
    root Element // sentinel list element, only &root, root.prev, and root.next are used
    len  int     // current list length excluding (this) sentinel element
}
type Element struct {
    next, prev *Element
    list *List
    Value interface{}
}
```

创建一个空链表:

```text
func New() *List { return new(List).Init() }
func (l *List) Init() *List {
    l.root.next = &l.root
    l.root.prev = &l.root
    l.len = 0
    return l
}
```

在链表表头插入数据:

```text
func (l *List) PushFront(v interface{}) *Element {
    l.lazyInit()
    return l.insertValue(v, &l.root)
}
// 如果链表为nil，初始化链表
func (l *List) lazyInit() {
    if l.root.next == nil {
        l.Init()
    }
}
func (l *List) insertValue(v interface{}, at *Element) *Element {
    return l.insert(&Element{Value: v}, at)
}
func (l *List) insert(e, at *Element) *Element {
  //将e插到root之后
    n := at.next 
    at.next = e
    e.prev = at
    //将原来root之后的Element插到e之后
    e.next = n
    n.prev = e 
    //声明e所处的list，并将len++
    e.list = l
    l.len++
    return e
}
```

在链表表尾插入数据：

```text
func (l *List) PushBack(v interface{}) *Element {
    l.lazyInit()
    //原理同上，即在root之前添加Element。其实在这儿我有个疑惑，为什么是在l.root.prev之后查呢，这实际上是因为l.root.prev指向的是最后一个节点
    return l.insertValue(v, l.root.prev)
}
func (l *List) insertValue(v interface{}, at *Element) *Element {
    return l.insert(&Element{Value: v}, at)
}
func (l *List) insert(e, at *Element) *Element {
  //将e插到root之后
    n := at.next
    at.next = e
    e.prev = at
    //将原来root之后的Element插到e之后
    e.next = n
    n.prev = e
    //声明e所处的list，并将len++
    e.list = l
    l.len++
    return e
}
```

在某个元素之前插入数据

```text
func (l *List) InsertBefore(v interface{}, mark *Element) *Element {
    if mark.list != l {
        return nil
    }
    return l.insertValue(v, mark.prev)
}
```

在某个元素之后插入数据

```text
func (l *List) InsertAfter(v interface{}, mark *Element) *Element {
    if mark.list != l {
        return nil
    }
    return l.insertValue(v, mark)
}
```

返回链表中的下一个元素或者 nil:

```text
func (e *Element) Next() *Element {
   if p := e.next; e.list != nil && p != &e.list.root {
      return p
   }
   return nil
}
```

返回链表中的前一个元素或者 nil :

```text
func (e *Element) Prev() *Element {
    if p := e.prev; e.list != nil && p != &e.list.root {
        return p
    }
    return nil
}
```

返回链表包含的元素数量。 该方法的复杂度为 O\(1\)

```text
func (l *List) Len() int { return l.len }
```

返回链表的第一个元素或者 nil

```text
func (l *List) Front() *Element {
    if l.len == 0 {
        return nil
    }
    return l.root.next
}
```

返回链表的最后一个元素或者 nil

```text
func (l *List) Back() *Element {
    if l.len == 0 {
        return nil
    }
    return l.root.prev
}
```

删除某个元素，如果 e 是链表 l 中的元素， 那么移除元素 e 。这里有个细节是把要删除的节点的prev指针和next指针指向nil以防止内存泄露。

```text
func (l *List) Remove(e *Element) interface{} {
    if e.list == l {
        // if e.list == l, l must have been initialized when e was inserted
        // in l or l == nil (e is a zero Element) and l.remove will crash
        l.remove(e)
    }
    return e.Value
}
func (l *List) remove(e *Element) *Element {
    e.prev.next = e.next
    e.next.prev = e.prev
    e.next = nil // avoid memory leaks
    e.prev = nil // avoid memory leaks
    e.list = nil
    l.len--
    return e
}
```

将元素 e 移动到链表的开头。

```text
func (l *List) MoveToFront(e *Element) {
    if e.list != l || l.root.next == e {
        return
    }
    l.move(e, &l.root)//将e移动到root之后，即表头
}
func (l *List) move(e, at *Element) *Element {
    if e == at {
        return e
    }
    //将e从链表中删除
    e.prev.next = e.next
    e.next.prev = e.prev
    // 将e指向root和root.next
    n := at.next
    at.next = e
    e.prev = at
    e.next = n
    n.prev = e

    return e
}
```

将元素 e 移动到链表的末尾

```text
func (l *List) MoveToBack(e *Element) {
    if e.list != l || l.root.prev == e {
        return
    }
    l.move(e, l.root.prev)
}
```

将元素 e 移动至元素 mark 之前

```text
func (l *List) MoveBefore(e, mark *Element) {
    if e.list != l || e == mark || mark.list != l {
        return
    }
    l.move(e, mark.prev)
}
```

将元素 e 移动到链表的末尾

```text
func (l *List) MoveAfter(e, mark *Element) {
    if e.list != l || e == mark || mark.list != l {
        return
    }
    l.move(e, mark)
}
```

将链表 other 的副本插入到链表 l 的末尾。 other 和 l 可以是同一个链表。

```text
func (l *List) PushBackList(other *List) {//实现原理与PushBack相类似，不过是采用循环的方式不断插入到链表的末尾
    l.lazyInit()
    for i, e := other.Len(), other.Front(); i > 0; i, e = i-1, e.Next() {
        l.insertValue(e.Value, l.root.prev)
    }
}
```

将链表 other 的副本插入到链表 l 的开头。 other 和 l 可以是同一个链表。

```text
func (l *List) PushFrontList(other *List) {
    l.lazyInit()
    for i, e := other.Len(), other.Back(); i > 0; i, e = i-1, e.Prev() {
        l.insertValue(e.Value, &l.root)
    }
}
```

### 总结 <a id="h-1"></a>

List的源码十分简单，但是其中还是有一些需要注意的点的。

1.在删除节点的时候一定要将节点的prev指针和next指针指向nil以防止内存泄露。

2.List的设计上有一个非常精妙的点，它使用了一个root节点作为链表第一个节点并将其prev指针指向了链表的最后一个节点，以达到对链表最后一个节点相关的操作实现时间复杂度o\(1\)，而不是o\(n\)

