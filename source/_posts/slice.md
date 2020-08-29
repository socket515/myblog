---
title: golang slice
date: 2020-08-29 11:21:01
tags:
 - golang
categories:
 - golang
---
<meta name="referrer" content="no-referrer" />
 slice是golang里常用的数据结构，是一个动态数组,可以随意地追加元素，而不用担心扩容问题。

## slice的内存结构
 slice是golang里的引用类型，与array区别，array是值类型。slice底层指向一个array，传递slice变量只是传递底层数组的引用
 
 底层结构
 ```go
    type slice struct {
    	Data  uintptr  // 底层数组的指针
    	Len   int      // 当前slice的长度
    	Cap   int      // 底层数组的容量
    }
 ```
 Data是底层数组的指针，Len代表当前slice的长度，Cap底层数组的容量。Len<=Cap,调用len时会返回Len,调用cap返回Cap。往slice追加元素时，Len小于Cap时候，直接追加在Data上，当Len>=Cap时候开始扩容复制Data里内容申请新的array。
 
 使用内存构造一个slice
 ```go
 array := [...]int{1, 2, 3}
 ptr := uintptr(unsafe.Pointer(&array))
 s := *(*[]int)(unsafe.Pointer(&reflect.SliceHeader{Data: ptr, Len: 3, Cap: 3}))
 ```
 
## 构造slice
 Go 语言中的切片有三种初始化的方式：
 
 通过下标的方式获得数组或者切片的一部分；
 使用字面量初始化新的切片；
 使用关键字 make 创建切片：
 ```go
 s1 := arr[:]
 s1 = s1[:]
 s2 := []int{1, 2, 3}
 s3 := make([]int, 3)
 ```

### 使用下标
 使用下标方式构造slice时候，编译器会识别这个语法，生成中间代码。构造出对应的slice结构，填充底层数组，len和cap。
 
 当下标为[:]时,cap和len都等于原array和slice
 ```go
 s := new(Slice)
 s.Data = uintptr(&array)
 s.Len = len(array)
 s.Cap = cap(array)
 ```
 
 当下标为[s:e], s为array或slice的开始索引下标，e为结束的下标,会检查s和e的值是否合法(0<=s<=len(array)&&0<=e<=len(array)&&s<=e)
 s和e缺省时候，s=0，e=len(array)
 ```go
 slice := new(Slice)
 slice.Len = e - s
 slice.Cap = cap(array) - s
 ```

 当下标[s:e:c],s和e情况与上面相同，c代表cap的值大小，当然也会检查c是否合法不能大于原slice或array的cap

### 字面量
 当我们使用字面量 []int{1, 2, 3} 创建新的切片时，会在编译期间将它展开成如下所示的代码片段：
 ```go
 var vstat [3]int
 vstat[0] = 1
 vstat[1] = 2
 vstat[2] = 3
 var vauto *[3]int = new([3]int)
 *vauto = vstat
 slice := vauto[:]
 ```
 1.根据切片中的元素数量对底层数组的大小进行推断并创建一个数组；
 2.将这些字面量元素存储到初始化的数组中；
 3.创建一个同样指向 [3]int 类型的数组指针；
 4.将静态存储区的数组 vstat 赋值给 vauto 指针所在的地址；
 5.通过 [:] 操作获取一个底层使用 vauto 的切片；

### 关键字
 使用make关键字在编译期间会检查参数，是否有传递cap，然后判断写入的参数合法性，若写入参数是变量则使用makeslice函数由运行时判断。
 
 make函数实现
 ```go
  func makeslice(et *_type, len, cap_ int) slice {
			// 根据切片的数据类型，获取切片的最大容量
			maxElements := maxSliceCap(et.size)
			// 比较切片的长度，长度值域应该在[0,maxElements]之间
			if len < 0 || uintptr(len) > maxElements {
				panic(errorString("makeslice: len out of range"))
			}
			// 比较切片的容量，容量值域应该在[len,maxElements]之间
			if cap < len || uintptr(cap) > maxElements {
				panic(errorString("makeslice: cap out of range"))
			}
			// 根据切片的容量申请内存
			p := mallocgc(et.size*uintptr(cap), et, true)
			// 返回申请好内存的切片的首地址
			return slice{p, len, cap}
  }
 ```

## 扩容
 在append的时候，增加len导致len>cap的时候，会触发扩容，append时候会计算这次要扩容大小，cap=oldcap+新追加内容长度。append是可以追加slice的。
 
 扩容策略
 每次扩容时候会先计算要扩容的大小cap, 比如原cap为1的切片,append一个长度为10的切片, cap=11. 计算最终扩充容量newcap， newcap >= cap·
 1. 首先判断，原来切片cap的两倍doublecap与cap的大小，若cap大则扩充的容量就是cap, newcap = cap·
 2. 若原cap<1024，newcap = doublecap
 3. 若原cap>=1024, 做一个循环  newcap = oldcap; for (newcap < cap) {newcap = newcap + newcap / 4}
 
 slice扩容后底层的数组会改变，旧的值会复制到新的底层数组中，并接触对旧数组的引用(等gc回收),这里会有一个新旧slice的问题。
 
 1. 未扩容
 ```go
	array2 := [4]int{10, 20, 30, 40}
	slice = array2[0:2]
	newSlice = append(slice, 50)
	fmt.Printf("Before slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	fmt.Printf("Before newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
	newSlice[1] += 10
	fmt.Printf("After slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	fmt.Printf("After newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
	fmt.Printf("After array = %v\n", array2)
 ```
 这个时候slice，由于newSlice，append未造成扩容，所以会改变slice的底层数组的值，因为两个slice是引用同一个底层数组
 
 2. 已扩容
 ```go
	array2 := [4]int{10, 20, 30, 40}
	slice := array2[2:4]
	newSlice := append(slice, 50)
	fmt.Printf("Before slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	fmt.Printf("Before newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
	newSlice[1] += 10
	fmt.Printf("After slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	fmt.Printf("After newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
	fmt.Printf("After array = %v\n", array2)
	fmt.Println()
 ```
 newslice已经扩所以不会影响原来的slice
 
## 小结
 切片的很多功能都是在运行时实现的了，无论是初始化切片，还是对切片进行追加或扩容都需要运行时的支持，需要注意的是在遇到大切片扩容或者复制时可能会发生大规模的内存拷贝，一定要在使用时减少这种情况的发生避免对程序的性能造成影响。
