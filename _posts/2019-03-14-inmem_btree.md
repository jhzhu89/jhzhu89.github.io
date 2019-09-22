---
layout: single
title:  "etcd mvcc package中的btree"
categories:
  - Written in Chinese
  - data structure
tags:
  - etcd
  - btree
toc: true
excerpt_separator: <!--more-->
---

本文记录对google开源的golang版本的B树的实现的分析。该B树的实现，被用在etcd中，来做内存索引。

<!--more-->

以下是摘自wikipedia的B树的定义：

> In [computer science](https://en.wikipedia.org/wiki/Computer_science), a **B-tree** is a self-balancing [tree data structure](https://en.wikipedia.org/wiki/Tree_data_structure) that maintains sorted data and allows searches, sequential access, insertions, and deletions in [logarithmic time](https://en.wikipedia.org/wiki/Logarithmic_time). The B-tree is a generalization of a [binary search tree](https://en.wikipedia.org/wiki/Binary_search_tree) in that a node can have more than two children.[[1\]](https://en.wikipedia.org/wiki/B-tree#cite_note-Comer-1) Unlike [self-balancing binary search trees](https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree), the B-tree is well suited for storage systems that read and write relatively large blocks of data, such as discs. It is commonly used in [databases](https://en.wikipedia.org/wiki/Database) and [file systems](https://en.wikipedia.org/wiki/File_system).

Knuth对B树的定义：

> a B-tree of order *m* is a tree which satisfies the following properties:
>
> 1. Every node has at most *m* children.
> 2. Every non-leaf node (except root) has at least ⌈*m*/2⌉ child nodes.
> 3. The root has at least two children if it is not a leaf node.
> 4. A non-leaf node with *k* children contains *k* − 1 keys.
> 5. All leaves appear in the same level.

B树非常适合用来做数据库索引。

etcd mvcc的实现中，使用了B树，来做内存中的索引，建立了key到revision之间的映射（bbolt中存储的是`revision->mvccpb.KeyValue`）。这一块具体的实现，可以参考`mvcc/{index.go, kvstore_txn.go}`等。

本文主要来分析etcd使用的[google开源的B树](https://github.com/google/btree)实现。

## Google golang b tree

我们可以在github上找到该golang版本B树的实现：https://github.com/google/btree。该实现，使用了`copy-on-write`技术，来减少数据拷贝。

### 目录结构

整个实现很简洁，包含实现代码、UT和一个benchmark测试。

```bash
tree ./btree
./btree
├── btree.go
├── btree_mem.go
├── btree_test.go
├── LICENSE
└── README.md

0 directories, 5 files
```

### 主要的数据结构

#### node

```go
// node is an internal node in a tree.
//
// It must at all times maintain the invariant that either
//   * len(children) == 0, len(items) unconstrained
//   * len(children) == len(items) + 1
type node struct {
	items    items
	children children
	cow      *copyOnWriteContext
}

// children stores child nodes in a node.
type children []*node

// items stores items in a node.
type items []Item

// Item represents a single object in the tree.
type Item interface {
	// Less tests whether the current item is less than the given argument.
	//
	// This must provide a strict weak ordering.
	// If !a.Less(b) && !b.Less(a), we treat this to mean a == b (i.e. we can only
	// hold one of either a or b in the tree).
	Less(than Item) bool
}
```

node是btree的节点。node存储一些元素，同时存储子节点的指针。以上定义较为清晰，只有cow，目前还不清楚具体是什么和可以用来做什么样的事情。

#### copyOnWriteContext

```go
// copyOnWriteContext pointers determine node ownership... a tree with a write
// context equivalent to a node's write context is allowed to modify that node.
// A tree whose write context does not match a node's is not allowed to modify
// it, and must create a new, writable copy (IE: it's a Clone).
//
// When doing any write operation, we maintain the invariant that the current
// node's context is equal to the context of the tree that requested the write.
// We do this by, before we descend into any node, creating a copy with the
// correct context if the contexts don't match.
//
// Since the node we're currently visiting on any write has the requesting
// tree's context, that node is modifiable in place.  Children of that node may
// not share context, but before we descend into them, we'll make a mutable
// copy.
type copyOnWriteContext struct {
	freelist *FreeList
}

// FreeList represents a free list of btree nodes. By default each
// BTree has its own FreeList, but multiple BTrees can share the same
// FreeList.
// Two Btrees using the same freelist are safe for concurrent write access.
type FreeList struct {
	mu       sync.Mutex
	freelist []*node
}
```

cow context存储了FreeList的指针。从注释的字面意思来看，copyOnWriteContext类型的指针决定node的所有权，一个拥有和某个node相同cow context的树才被允许修改那个node。那么如何通过cow指针来确定node是属于那个btree呢？在实现中，主要是通过判断btree和node的cow指针是否相等，来检查某个node是否属于某个btree。

FreeList存储了一些尚未被使用的node。FreeList可以一定程度上减少创建、销毁node的开销。在创建btree的时候，我们可以指定提前创建这样一个free node pool。随着btree的使用，可能有些node的元素会被删除光，这些node便会被重新放回freelist中，等待将来的使用。注意，这个freelist的大小在创建的时候就被固定了，之后无法被更改。

#### BTree

```go
// BTree is an implementation of a B-Tree.
//
// BTree stores Item instances in an ordered structure, allowing easy insertion,
// removal, and iteration.
//
// Write operations are not safe for concurrent mutation by multiple
// goroutines, but Read operations are.
type BTree struct {
	degree int
	length int
	root   *node
	cow    *copyOnWriteContext
}
```

length用来记录BTree中元素的数量。Knuth的定义中，并没有degree。该实现中，作者采用的是另外一种定义。这些不同的定义只是从不同的角度描述了B树的性质，并不是说B树本身不同。这里摘取了degree的定义如下，完整的定义可以参考链接：

> There are lower and upper bounds on the number of keys a node can contain. These bounds can be expressed in terms of a fixed integer t >= 2 called the minimum degree of the B-tree:
>
> - Every node other than the root must have at least *t*-1 keys. Every internal node other than the root thus has at least *t* children. If the tree is nonempty, the root must have at least one key.
> - Every node can contain at most 2*t*-1 keys. Therefore, an internal node can have at most 2*t* children. We say that a node is *full* if it contains exactly 2*t*-1 keys.
>
> http://www.cs.utexas.edu/users/djimenez/utsa/cs3343/lecture16.html

根据degree的定义可以看出，degree可以决定一个node 元素数量和孩子节点数量的最大和最小值。BTree中自然需要保存root节点用以访问整个B树。cow用来判断某个节点是否属于这棵B树。

该实现中用到的数据结构已经介绍完毕，接下来，我们来分析一下B树常用操作的具体实现。

### BTree相关的方法和函数

```go
▼+BTree : struct
    [methods]
   +Ascend(iterator ItemIterator)
   +AscendGreaterOrEqual(pivot Item, iterator ItemIterator)
   +AscendLessThan(pivot Item, iterator ItemIterator)
   +AscendRange(greaterOrEqual, lessThan Item, iterator ItemIterator)
   +Clear(addNodesToFreelist bool)
   +Clone() : *BTree
   +Delete(item Item) : Item
   +DeleteMax() : Item
   +DeleteMin() : Item
   +Descend(iterator ItemIterator)
   +DescendGreaterThan(pivot Item, iterator ItemIterator)
   +DescendLessOrEqual(pivot Item, iterator ItemIterator)
   +DescendRange(lessOrEqual, greaterThan Item, iterator ItemIterator)
   +Get(key Item) : Item
   +Has(key Item) : bool
   +Len() : int
   +Max() : Item
   +Min() : Item
   +ReplaceOrInsert(item Item) : Item
    [functions]
   +New(degree int) : *BTree
   +NewWithFreeList(degree int, f *FreeList) : *BTree
```

以上是BTree暴露的所有接口。BTree中没有定义unexported方法。从调用者的角度来看，这些接口比较清晰简单，主要方便调用者来向B树中插入数据，或删除数据，同时暴露了一些迭代操作。接下来，我们从这些接口入手，由外向内来剖析其实现细节。

#### func (t *BTree) ReplaceOrInsert(item Item) Item

顾名思义，如果item不存在，则这个函数会将这个item插入到B树中;如果B树中已经存在一个item，和这个待插入的item相同（!item.Less(item_in_btree) && !item_in_btree.Less(item)），则这个函数会用这个item去替换旧的item，并将旧的item返回给调用者。接下来，我们来分析具体实现。

```go
// ReplaceOrInsert adds the given item to the tree.  If an item in the tree
// already equals the given one, it is removed from the tree and returned.
// Otherwise, nil is returned.
//
// nil cannot be added to the tree (will panic).
func (t *BTree) ReplaceOrInsert(item Item) Item {
	if item == nil {
		panic("nil item being added to BTree")
	}
	if t.root == nil {
		t.root = t.cow.newNode()
		t.root.items = append(t.root.items, item)
		t.length++
		return nil
	} else {
		t.root = t.root.mutableFor(t.cow)
		if len(t.root.items) >= t.maxItems() {
			item2, second := t.root.split(t.maxItems() / 2)
			oldroot := t.root
			t.root = t.cow.newNode()
			t.root.items = append(t.root.items, item2)
			t.root.children = append(t.root.children, oldroot, second)
		}
	}
	out := t.root.insert(item, t.maxItems())
	if out == nil {
		t.length++
	}
	return out
}
```

该函数首先检查了item的有效性，如果无效，则直接panic。就是说，调用者应该保证传入参数的有效性。

接下来，如果这是一颗空树，则创建一个新的node，并将这个item插入到node中。让我们step in到cow.newNode()中一探究竟。

##### func (c *copyOnWriteContext) newNode() (n *node) 

```go
func (c *copyOnWriteContext) newNode() (n *node) {
	n = c.freelist.newNode()
	n.cow = c
	return
}

func (f *FreeList) newNode() (n *node) {
	f.mu.Lock()
	index := len(f.freelist) - 1
	if index < 0 {
		f.mu.Unlock()
		return new(node)
	}
	n = f.freelist[index]
	f.freelist[index] = nil
	f.freelist = f.freelist[:index]
	f.mu.Unlock()
	return
}
```

cow.newNode()尝试从freelist的尾部获取一个node来使用。如果freelist已经是空的了，才会new一个node。这样做的目的是一定程度减少node的分配和释放来节省内存操作的开销。主语FreeList.newNode()中的锁操作，这把锁主要是针对不同B树共享一个FreeList的情况，来同步他们对FreeList的并发操作。

##### func (n *node) mutableFor(cow *copyOnWriteContext) *node

如果这颗B树非空，copy-on-write机制将开始发挥作用，首先root将被克隆，修改操作会作用于克隆root。同时，这段代码将会检查root节点是否需要分裂，如果需要，则继续进行插入item前的准备工作。

```go
func (n *node) mutableFor(cow *copyOnWriteContext) *node {
	if n.cow == cow {
		return n
	}
	out := cow.newNode()
	if cap(out.items) >= len(n.items) {
		out.items = out.items[:len(n.items)]
	} else {
		out.items = make(items, len(n.items), cap(n.items))
	}
	copy(out.items, n.items)
	// Copy children
	if cap(out.children) >= len(n.children) {
		out.children = out.children[:len(n.children)]
	} else {
		out.children = make(children, len(n.children), cap(n.children))
	}
	copy(out.children, n.children)
	return out
}
```

mutebaleFor()是cow机制起作用的地方。前文我们提到过，在该实现中，node结构中的cow被用来校验该node是否属于某棵B树。所以，如果n.cow == cow，则说明n属于拥有该cow的B树，则该B树可以直接修改n，因此，该函数直接返回了n。如果n不属于这个B树，则这段代码会从freelist中获取一个node，并且深拷贝n，然后返回这个新node。

##### func (n *node) split(i int) (Item, *node)

接着，如果当前root节点已经存储了超过maxItems()个元素，这将root从中间分裂。

```go
// split splits the given node at the given index.  The current node shrinks,
// and this function returns the item that existed at that index and a new node
// containing all items/children after it.
func (n *node) split(i int) (Item, *node) {
	item := n.items[i]
	next := n.cow.newNode()
	next.items = append(next.items, n.items[i+1:]...)
	n.items.truncate(i)
	if len(n.children) > 0 {
		next.children = append(next.children, n.children[i+1:]...)
		n.children.truncate(i + 1)
	}
	return item, next
}
```

split函数从指定的index，把n分裂成两个node，n缩小，其余的item和孩子被放进一个新的node。返回值是位于index的item，和新节点next。

旧的root分裂完成后，需要新创建一个root节点，并将分裂出来的两个node作为新root的孩子节点。经过以上步骤，就完成了插入item前的准备工作。

##### func (n *node) insert(item Item, maxItems int) Item

接下来执行item的插入。

```go
func (n *node) insert(item Item, maxItems int) Item {
	i, found := n.items.find(item)
	if found {
		out := n.items[i]
		n.items[i] = item
		return out
	}
	if len(n.children) == 0 {
		n.items.insertAt(i, item)
		return nil
	}
	if n.maybeSplitChild(i, maxItems) {
		inTree := n.items[i]
		switch {
		case item.Less(inTree):
			// no change, we want first split node
		case inTree.Less(item):
			i++ // we want second split node
		default:
			out := n.items[i]
			n.items[i] = item
			return out
		}
	}
	return n.mutableChild(i).insert(item, maxItems)
}
```

首先，在当前节点n的items中查找待插入的item，如果能够找到，则直接进行替换，并返回旧的item。如果没有找到这个item，find函数会返回这个item的插入位置i。如果n是一个叶节点，即没有任何孩子节点，则表明item应该被插入第i个位置，即调用insertAt(i, item)。

如果这是一个非叶节点，则在插入之前，先要检查第i个孩子节点是否需要分裂。如果第i个孩子存储的元素数量超过了最大值，则需要分裂成两个节点。

##### func (n *node) maybeSplitChild(i, maxItems int) bool

该函数将n分裂成均等的两个节点，然后把位于分裂点i的item在插入到分裂后的第一个节点，最后把他们再插入回n中。

```go
func (n *node) maybeSplitChild(i, maxItems int) bool {
	if len(n.children[i].items) < maxItems {
		return false
	}
	first := n.mutableChild(i)
	item, second := first.split(maxItems / 2)
	n.items.insertAt(i, item)
	n.children.insertAt(i+1, second)
	return true
}
```

由于当前在处理的是非叶节点，分裂完成后，需要继续检查是否要递归向孩子节点插入item，如果是，则递归调用第i个孩子节点的insert()方法，同时，也要注意调用n.mutableChild(i)来克隆一个孩子节点进行修改 。

至此，`ReplaceOrInsert`就分析完了。

##### 总结

- cow机制作用于从root一直到待插入item的孩子的路径上的所有节点
- 该函数尝试分裂root节点，创建一个新的有两个孩子的root‘，所以root的分裂会增加树的高度
- 该函数尝试分裂孩子节点，分裂成功后，会给当前节点新加入一个item和一个孩子节点

#### func (t *BTree) Get(key Item) Item

该函数在BTree中查找key，如果找到，则返回这个Item，如果找不到，则返回nil。

```go
// Get looks for the key item in the tree, returning it.  It returns nil if
// unable to find that item.
func (t *BTree) Get(key Item) Item {
	if t.root == nil {
		return nil
	}
	return t.root.get(key)
}
```

该函数实际调用了node的get方法，递归的进行查找。

```go
func (n *node) get(key Item) Item {
	i, found := n.items.find(key)
	if found {
		return n.items[i]
	} else if len(n.children) > 0 {
		return n.children[i].get(key)
	}
	return nil
}
```

这是一个直观的递归实现，在此不过多介绍。

#### func (t *BTree) Max() Item

该函数获取BTree中最大的元素。

```go
// Max returns the largest item in the tree, or nil if the tree is empty.
func (t *BTree) Max() Item {
	return max(t.root)
}

func max(n *node) Item {
	if n == nil {
		return nil
	}
	for len(n.children) > 0 {
		n = n.children[len(n.children)-1]
	}
	if len(n.items) == 0 {
		return nil
	}
	return n.items[len(n.items)-1]
}
```

实现较为直观，从根节点开始，一直向下寻找到最右端的孩子节点，然后返回它最大的元素。

#### func (t *BTree) Min() Item

该函数获取BTree中最小的元素。

```go
// Min returns the smallest item in the tree, or nil if the tree is empty.
func (t *BTree) Min() Item {
	return min(t.root)
}

func min(n *node) Item {
	if n == nil {
		return nil
	}
	for len(n.children) > 0 {
		n = n.children[0]
	}
	if len(n.items) == 0 {
		return nil
	}
	return n.items[0]
}
```

它是Max的镜像实现。

#### func (t *BTree) Delete(item Item) Item

该函数从BTree中删除指定的item。

```go
// Delete removes an item equal to the passed in item from the tree, returning
// it.  If no such item exists, returns nil.
func (t *BTree) Delete(item Item) Item {
	return t.deleteItem(item, removeItem)
}

func (t *BTree) deleteItem(item Item, typ toRemove) Item {
	if t.root == nil || len(t.root.items) == 0 {
		return nil
	}
	t.root = t.root.mutableFor(t.cow)
	out := t.root.remove(item, t.minItems(), typ)
	if len(t.root.items) == 0 && len(t.root.children) > 0 {
		oldroot := t.root
		t.root = t.root.children[0]
		t.cow.freeNode(oldroot)
	}
	if out != nil {
		t.length--
	}
	return out
}
```

### To be continued.
