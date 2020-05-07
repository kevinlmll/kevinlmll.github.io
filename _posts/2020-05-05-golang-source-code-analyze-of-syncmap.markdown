---
layout:     post
title:      "Golang里map和Sync.Map的不同"
subtitle:   "\"细品sync.Map源代码\""
date:       2020-05-05 12:00:00
author:     "kevinlmll"
header-img: "img/post-bg-2015.jpg"
tags:
    - golang
---

### Golang里map和Sync.Map的不同

使用Golang的同学都知道，golang提供了map和sync.Map两种字典数据结构。简单的从包名可以看出，sync.Map作为sync包里的一个数据类型，相对于内置的map数据结构，应该是支持同步语义，简化开发做同步化的编码过程。但是map+lock/mutex的方式也可以保证map里操作数据同步化，那么，这两者的差异在哪里呢？

我们先看一下sync.Map包的文档说明:
>Map is like a Go map[interface{}]interface{} but is safe for concurrent use by multiple goroutines without additional locking or coordination. Loads, stores, and deletes run in amortized constant time.

*sync.Map是一种类型Go map[interface{}]interface{}的数据结构，但是它在多协程并发使用上更加安全，并且不需要额外的锁和条件变量。它在读、写、删除的操作平均耗时是常量时间。*

>The Map type is specialized. Most code should use a plain Go map instead, with separate locking or coordination, for better type safety and to make it easier to maintain other invariants along with the map content.

*sync.Map这个数据结构是特制的。为了更好的类型安全，map里不变量的更易维护性，建议是使用原始的Go map加独立锁/条件变量的方式来开发*

>The Map type is optimized for two common use cases: (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex.

*sync.Map数据结构专门为以下两种场景下进行优化：（1）对于一些Key,关联的内容实体是只写一次，但有大量的读，类似，使用缓存时只有Key增长，或者（2）多个并发协议读、写、修改的Keys集合是不相交的。基于以上的两个场景，对比Go map+独立Mutex/RWMutex方案，使用sync.Map可以极大的减少争夺锁的消耗*

>The zero Map is empty and ready for use. A Map must not be copied after first use.`

*初始化后的sync.Map是空的，可以直接使用。当sync.Map被使用后，**一定**不能进行复制*

从文档的描述看到，sync.Map主要是针对两类场景做了优化：写少读多（写只是在初始化创建），以及多个协程变更的Key集不相交。那么具体实现的细节是怎么样的呢？

sync.Map涉及到的数据结构如图所示，简单介绍一下Map结构里几个字段的作用。mu与dirty配合使用，当需要写dirty里的数据时，通过mu来获取锁；read，主要是历史数据，读的时候使用，可以不加锁，更新历史key的数据时，需要用到mu来得到锁；misses，记录在read未获得key数据的次数，当该次数超过dirty的大小时，会将dirty的数据同步到read，同时清空dirty[含义就是：未命中cache，导致需要重读的消耗已经大于把dirty复制到read的消耗]。
![](/img/in-post/sync-map-struct.png)

先看写过程，sync.Map.Store接口负责数据更新，和sync.Map.Delete接口负责删除。
Store函数代码分析
```golang
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		e.storeLocked(&value)
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}
```
1. 加载*read*，判断key是否在read里，如果在read里，尝试写入，成功则结束（成功条件：key对应的value不是已删除，更新时，通过多次compare and swap保证原子性），否则转2；
2. 如果key不在read里或者key对应的value已删除，通过mu获取锁，重新加载*read*，重新判断key是否在read里（有可能在等待锁过程中，有其它数据更新），如果存在，判断value是否已删除过，若value已删除过，则在*dirty*增加key-value关系对；更新value新的值；若key不在read里，转3；
3. 判断key是否在dirty里，如果存在，更新value新的值；否则转4；
4. 如果*read*未修改过，创建dirty（分配内存，copy数据，**value为nil的key是处于被删除状态，不会被copy**），并更新read的状态为“被修改过”，在*dirty*增加key-value关系对；
5. 释放mu锁；

Delete函数代码分析
```golang
func (m *Map) Delete(key interface{}) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			delete(m.dirty, key)
		}
		m.mu.Unlock()
	}
	if ok {
		e.delete()
	}
}
```
1. 加载*read*，如果key已存在，直接删除，结束；否则转2；
2. 如果key不存在，并且read被修改过，通过mu获取锁，再重新加载*read*，再重新判断key是否在read里（有可能在最近一次修改过），如果key还是不存在并且read还是处于“被修改过”的状态，从*dirty*里删除key；释放锁；
3. 如果在是2里查到key是存在的，则直接删除，结束；

再看读过程，sync.Map.Load接口负责读取数据，sync.Map.Range接口负责遍历数据集
Load函数代码分析
```golang
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}
```
1. 加载*read*，判断key是否存在read里，如果不存在并且read处于“被修改”状态，申请mu锁；重新加载*read*，如果read还是没有key并且处理“被修改”
状态，从*dirty*里获取，释放锁；通过*missLocked*进行一次读取统计；释放锁；
2. 如果read里没有Key, 并且read没有"被修改"的状态，则直接返回(nil, false)
3. key存在，返回对应的value(此时，也有可能value为nil)；

Range函数代码分析
```golang
func (m *Map) Range(f func(key, value interface{}) bool) {
	// We need to be able to iterate over all of the keys that were already
	// present at the start of the call to Range.
	// If read.amended is false, then read.m satisfies that property without
	// requiring us to hold m.mu for a long time.
	read, _ := m.read.Load().(readOnly)
	if read.amended {
		// m.dirty contains keys not in read.m. Fortunately, Range is already O(N)
		// (assuming the caller does not break out early), so a call to Range
		// amortizes an entire copy of the map: we can promote the dirty copy
		// immediately!
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		if read.amended {
			read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}
```
1. 加载*read*，如果read处于"被修改"状态，这里，直接做一次从dirty到read的替换，更新read，并将dirty置为nil, miss次数重置为0；释放锁；
2. 从*read*里循环遍历key-value;

总结：*read*可以当作一个静态数据集，支持不加锁，快速的读取；*dirty*包括*read*里非nil的数据集和新增的数据集；当访问*dirty*里的key次数大于*dirty*大小时，会将*dirty*里的数据copy到*read*，减少性能消耗（因为能在*read*访问到key时，是可以不加锁操作。而访问*dirty*或者key不在read时[read被修改过]，需要加锁来保证数据一致性）。再回头看看sync.Map的适用场景：
>（1）对于一些Key,关联的内容实体是只写一次，但有大量的读，类似，使用缓存时只有Key增长

大量的读可以在read里获取到，基本可以不加锁，对比使用单独的mutex/rwmutex要高效得多。少量的写并不影响，只有在访问次数达到一定量后，才会做一次迁移；
>（2）多个并发协议读、写、修改的Keys集合是不相交的

同样的原因，由于keys不相交，大部分的操作可以在read实现不加锁处理，比使用单独的全局锁要高效。新增或者key在dirty时，无法避免全局锁。

P.S. 写这篇文章的时候，没有在网上google相关的资料，然后上传到公众号时，为了找个封面图，发现有人发过很详细的解读，图文并茂，这里附上链接：[sync_map](http://www.sreguide.com/go/sync_map.html)
