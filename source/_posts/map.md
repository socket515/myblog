---
title: golang map
date: 2020-09-06 11:18:37
tags:
- golang
categories:
- golang
---
<meta name="referrer" content="no-referrer" />
 哈希表是一个优秀的数据结构，能以O(1)的时间复杂度查找、增加和删除数据。提供了一种键和值的映射，实现这一个哈希表有两个关键点`哈希函数`和`冲突处理`，不同语言有不同的内置实现，map是golang中内置实现的哈希表。

## map的结构
 golang在`runtime`中实际上使用`hmap`结构来表示map，代码中的map都会在编译器转换成`hmap`
 
```go
type hmap struct {
	count     int
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32

	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr

	extra *mapextra
}
```
 1. count代表map中的元素个数
 2. flags代表当前的状态(用某个bit位代表某个状态如第三个bit位为1代表map正在写入)
 3. B表示`buckets`的数量，`buckets`的数量是2的`B`次方个
 4. hash0 是哈希的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入
 5. `buckets`是存一个`bmap`数组的指针，存放桶的地址。其中`oldbuckets`代表扩容前的`buckets`

 `hmap`的桶就是`bmap`，一个`bmap`存放8个键值对，当超过8个时候就会使用`extra.overflow`中的溢出桶中，由此可知golang map处理冲突的方式是拉链法。
 
```go
type bmap struct {
	tophash [bucketCnt]uint8
}
```
 上面的这个是`bmap`的结构体，只存储了长度为8(bucketCnt=8)的uint8数组。`tophash`表示是桶中8个key的前8位哈希，先比较前8位哈希用于快速查找桶中元素。
 
 `bmap`结构体实际上并不只有`tophash`属性，因为keySize和valueSize不定,实际上获取桶中其他值通过指针运算。`bmap`实际结构如下:
```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```
 当`bmap`存储的数据超过8个时候会使用到`overflow`指针指向的溢出桶，不过过多的溢出桶也会造成`hmap`的扩容。

## map的初始化
 跟之前的slice一样，map初始化一样有字面量初始化和运行时初始化。
 
### 字面量
 字面量初始化如下:
```go
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```
 当初始化的数量小于25时候编译器会将上述代码处理成
```go
hash := make(map[string]int, 3)
hash["1"] = 2
hash["3"] = 4
hash["5"] = 6
```
 当初始化的数量大于25时，会使用slice来帮助处理，如下:
```go
hash := make(map[string]int, 26)
vstatk := []string{"1", "2", "3", ... ， "26"}
vstatv := []int{1, 2, 3, ... , 26}
for i := 0; i < len(vstak); i++ {
    hash[vstatk[i]] = vstatv[i]
}
```

### 运行时
 使用`make`函数初始化map时候，会调用`runtime.makemap`函数来构造map。

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```
 这个函数执行分为下面几个部分
 1. 计算哈希占用的内存是否溢出或者超出能分配的最大值
 2. 调用`fastrand`获取一个随机的哈希种子
 3. 根据传入的 hint 计算出需要的最小需要的桶的数量, hint小于8时候，只在赋值才申请或扩容桶
 4. 使用`runtime.makeBucketArray`创建用于保存桶的数组

 `runtime.makeBucketArray`函数根据传入的`B`计算出的需要创建的桶数量在内存中分配一片连续的空间用于存储数据
```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
 当`B`小于4的时候，由于数据比较小，使用溢出桶的可能性较低，这个时候会省略创建的消耗不创造溢出桶。当B的数量大于4的时候才创造溢出桶，溢出桶的数量等于2的`B`-4次方。我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 hmap 中的不同字段引用。

## 读取map
 golang中读取map有两种情况: 
```go
 v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
 v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
```
 区别在于返回一个值和返回两个值，在编译时都会改造成调用对应的函数，`mapaccess1`和`mapaccess2`的区别在于是否有返回这个值是否存在。下面主要讲`mapaccess1`

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // map为空时候直接返回对应值的0值
 	if h == nil || h.count == 0 {
 		if t.hashMightPanic() {
 			t.hasher(key, 0) // see issue 23734
 		}
 		return unsafe.Pointer(&zeroVal[0])
 	}
    // map正在写直接抛出异常
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
    // 计算哈希值
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
    // 通过哈希找到对应的桶
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // 判断map是否在扩容
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 对应的桶是否已经在新桶中，如果不是取旧桶
		if !evacuated(oldb) {
			b = oldb
		}
	}
    // 获取哈希的前8位用于在桶中快速找值
	top := tophash(hash)
bucketloop:
    // 在桶中找不到去溢出桶中找
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
            //比对哈希前8位，不相同跳过
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
            // 比对键值是否一样
			if t.key.equal(key, k) {
                // 找到了取值返回
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
    // 找不到返回0值
	return unsafe.Pointer(&zeroVal[0])
}
```
 mapaccess1大致完成下面操作
 1. 判断空map直接返回0值
 2. 判断是否在写，在写直接panic，golang的map 不是线程安全的结构体，多线程读写需要外部加锁或使用`sync.Map`
 3. 计算哈希找出对应的桶，这时候会判断是否在扩容，若扩容则判断对应桶是否已经迁移，若无迁移则取旧桶。
 4. bucketloop中遍历桶和溢出桶，然后比对每一个桶tophash，相同再比对key。每一个桶都存储键对应的 tophash，每一次读写操作都会与桶中所有的 tophash 进行比较，用于选择桶序号的是哈希的最低几位，而用于加速访问的是哈希的高 8 位，这种设计能够减少同一个桶中有大量相等 tophash 的概率。
 
## 写入map
 如`hash[k]=3`，当map出现在符合的左侧时候，编译时会转为调用`runtime.mapassign`,此函数和`runtime.mapaccess1`相似。
 
 1.首先根据key计算出哈希和桶，然后设置flags，表示当前正在写，在计算哈希前会判断当前是否在写，避免并发写直接panic。若当前桶为空会调用`newobject`创造新桶。
```go
    if h.flags&hashWriting != 0 {
            throw("concurrent map writes")
    }
    hash := t.hasher(key, uintptr(h.hash0))
    // 记录正在写入
	h.flags ^= hashWriting
    if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}
```
 2. 判断`hmap`是否在扩容中，然后再判断对应的旧桶是否迁移了，若无则先迁移旧桶。golang map扩容是渐进的，将扩容操作分散到每一个写操作中。扩容操作后面详细讲。
```go
again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)
```
 3. 然后遍历桶和桶中的`tophash`找出对应键值对。`inserti`表示目标元素的在桶中的索引，`insertk`和`val`分别表示键值对的地址，获得目标地址之后会直接通过算术计算进行寻址获得键值对`k`和`val`
```go
var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if !alg.equal(key, k) {
				continue
			}
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
```
 4. 判断是否需要扩容，一般两种情况会触发扩容，装载因子超过0.65或使用过多溢出桶
```go
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
```
 5. 哈希遍历溢出桶，这个时候找不到地方存放值了因为桶已经满了或已有溢出桶也满，需要构建新的溢出桶。
```go
	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++
```
 6. 收尾操作，写标记位置0，返回元素
```go
done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
```

## map扩容
 在写操作和读操作都接触了扩容，已经知道了扩容的条件:
 1. 装载因子已经超过 6.5
 2. 哈希使用了太多溢出桶
 
 golang map的扩容并不是需要扩容的时候就开始将全部桶迁移的，是扩散到每一个写或删操作中。

 有一种特殊的扩容情况，叫等量扩容。当我们持续向哈希中插入数据并将它们全部删除时，如果哈希表中的数据量没有超过阈值，就会不断积累溢出桶造成缓慢的内存泄漏。`sameSizeGrow`通过重用已有的哈希扩容机制，一旦哈希中出现了过多的溢出桶，它就会创建新桶保存数据，垃圾回收会清理老的溢出桶并释放内存
 
 扩容的入口函数是`runtime.hashGrow`，如下可以看出扩容大小每次扩容两倍，并设置扩容标记和新建新桶设置旧桶。
```go
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
    // 增加B值
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	h.extra.oldoverflow = h.extra.overflow
	h.extra.overflow = nil
	h.extra.nextOverflow = nextOverflow
}
```
 哈希在扩容的过程中会通过`runtime.makeBucketArray`创建一组新桶和预创建的溢出桶，随后将原有的桶数组设置到`oldbuckets`上并将新的空桶设置到`buckets`上，溢出桶也使用了相同的逻辑进行更新
 
 `runtime.hashGrow`只是设置了标记和创建新桶，并没有正在地迁移数据，迁移数据的执行函数是`runtime.evacuate`,会对传入的桶再分配。
```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    // 找出旧桶
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	// 判断旧桶是否已经迁移了
	// 无迁移会进行迁移
	if !evacuated(b) {
        // 创建evacDst
        // 会将旧桶的数据分流到两个新桶中。
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))

		y := &xy[1]
		y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
		y.k = add(unsafe.Pointer(y.b), dataOffset)
		y.v = add(y.k, bucketCnt*uintptr(t.keysize))
  }
}
```
 `runtime.evacuate`最后会调用`runtime.advanceEvacuationMark`增加哈希的`nevacuate`计数器，在所有的旧桶都被分流后清空哈希的`oldbuckets`和`oldoverflow`字段：
```go
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		h.oldbuckets = nil
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```

## map的删除
 如果想要删除哈希中的元素，就需要使用`delete`关键字，这个关键字的唯一作用就是将某一个键对应的元素从哈希表中删除，无论是该键对应的值是否存在，这个内建的函数都不会返回任何的结果。
 
 编译时会将`delete`操作转为调用`mapdelete`函数，map的删除与写入相似，都是检查扩容并迁移和找对对应元素。

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	...
	if h.growing() {
		growWork(t, h, bucket)
	}
	...
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if !alg.equal(key, k2) {
				continue
			}
			*(*unsafe.Pointer)(k) = nil
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			*(*unsafe.Pointer)(v) = nil
			b.tophash[i] = emptyOne
			...
		}
	}
}
```

## 小结
 1. golang map使用拉链法解决冲突
 2. 读取、写入和删除在编译其都会转换成对应的函数
 3. golang map扩容会分散到每一个写或删操作中
 4. 使用哈希高8位快速定位桶中的元素。

