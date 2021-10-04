---
title: json序列化和反序列化优化
date: 2021-08-08 21:46:08
tags:
 - golang
 - json
categories:
 - golang
---
<meta name="referrer" content="no-referrer" />
  
  有关这次json序列化和反序列化优化
  - 根据本人优化json序列化和反序列化的心路历程编写
  - 受限个人水平问题, 主要讲golang中json相关优化思路
  - 提供一些其他场景优化的思路
  - json简单介绍
  - golang: unsafe, reflect

# json

  json是易于人的读和写的轻量文本交互格式，由六个标记字符(`[`,`]`,`{`,`}`,`:`,`,`), 字符串,数字,三个名称(true,false,null)组成.

 - key必需是字符串类型
 - val必需是数字、字符串、对象、数组，或以下三个文字名称(true,false,null)之一
 - 对象：对象结构表示为围绕零或多个名称/值对(或多个成员)的一对花括号
 - 数组: 数组结构表示为围绕零个或多个值(或多个元素)的方括号。元素用逗号分隔。
 - 数字: 数字要以10进制形式表示，指数部分可以以E或e开头,分数部分跟随小数点,不允许使用Infinity和NaN表示
 - 字符串：由双引号包括，字符串中包括非转义字符，需要转义字符(引号、反斜线分隔符\和控制字符(U+0000到U+001 F))需要转义(前面+\),字符编码默认为utf-8, 对于utf-16字符的处理\u后面根4个16进制数字。
 - 字节数组: json RFC 8259标准并没有提到关于bytes的处理，由于json是可读的，一般是把bytes base64编码后当string存储。

# golang标准库json

  golang从1.1到1.15,标准库做到优化有: 1.根据reflect.Type缓存encode和decode方法 2. 使用sync.Pool避免对象频繁创建. 从下面火焰图来看序列化和反序列化最大问题是反射。

 > 标准库json marshal cpu火焰图
 ![](json/json_marshal.jpg)

 > 标准库json unmarshal cpu火焰图
 ![](json/json_unmarshal.jpg)

# 反射与unsafe

  golang放射本质对interface的拆解, 是区分type和value。reflect包就是针对interface这两个值分别操作。
 ```go
 type emptyInterface struct {
 	typ  unsafe.Pointer
 	word unsafe.Pointer
 }
 ```

json序列化和反序列化操作中，需要频繁地创建对象`reflect.New(typ)`,然后调用`val.field(i).set(reflect.Valueof(itface))`或`val.field(i).set(reflect.New(xxx)))`, 针对new场景和set场景做benchmark

`unsafe`包含一些指针操作，可以使用计算指针的偏移量来进行赋值。在获取到reflectType的情况下知道每个属性的偏移量可以使用指针计算进行赋值。

```go
var offset1 uintptr

func init()  {
	typ := reflect.TypeOf(User{})
	offset1 = typ.Field(1).Offset
}

func setFunc()  {
	u := User{
		ID: 18,
		Name: "hhh",
	}
	t := unsafe.Pointer(&u)
	*(*int)(t) = 19
	*(*string)(unsafe.Pointer(uintptr(t) + offset1)) = "hello"
	fmt.Println(u) // {19, hello}
}
```

# 一个另类的优化思路

 在一些时候，某些struct往往只有一个或较少的field会改动,比如在igc中聊天消息推送只会改变`LobbyResponse`.`LobbyResponseBody`.`Payload`,这个值,可以提前序列化好这个bytes,通过拼接方法完成序列化.
 
```
goos: darwin
goarch: amd64
BenchmarkMarshalJsonBytes-4              9791989               112 ns/op
BenchmarkMarshalJson-4                   1329961               896 ns/op
PASS
ok      command-line-arguments  9.843s
```

 在RFC 6902中规范中提出JSONPATH,允许一些协议可以合并两个json或一些操作为json增加field
 
 - [RFC 6902](https://tools.ietf.org/html/rfc6902)
 - [go json-path](https://github.com/evanphx/json-patch)
 
 相对应，在反序列化时候，往往只需要读取某一个值，这样没必要做序列化，可以从json中获取某些值。
 
 - [fastjson](https://github.com/valyala/fastjson)
 - [gjson](https://github.com/tidwall/gjson)

# 实现自定义json接口

  实现标准库提供的两个接口
```go
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
	UnmarshalJSON([]byte) error
}
```

实现接口时候，调用json库方法, 会导致死循环。
```go
func(u *User) MarshalJSON() ([]byte, error) {
  return json.Marshal(u)
}

// 1. 使用新的struct
func(u *User) MarshalJSON() ([]byte, error) {
  return json.Marshal(sturct{ xxxx } { u.xxx})
}

// 2. 直接替换typ
// 先生成typ
fileds = reflect.TypeOf(User{}).Fileds(xxxx)
typ := reflect.StructOf(fileds)
typPtr := (*emptyInterface)(unsafe.Pointer(&typ)).word
func(u *User) MarshalJSON() ([]byte, error) {
  var v interface{}
  v = u
  e := (*emptyInterface)(unsafe.Pointer(&v))
  e.typ = typPtr
  return json.Marshal(v)
}
```

# 用代码生成方式避免reflect

 使用语法分析工具，分析struct，为struct生成`MarshalJSON`方法和`UnmarshalJSON`方法。
 
 - [easyjson](https://github.com/mailru/easyjson)
 - [ffjson](https://github.com/pquerna/ffjson)
 
 缺点:
 - 需要每次都更新生成方法(可以ci解决)
 
# 不使用代码生成解决方法

 序列化和反序列化需要解决的两个主要问题:
 - 需要序列化的类型
 - 为字段赋值问题
 
 解决方案
 - runtime保证type的指针唯一性
 - 使用unsafe做指针运算

 - [gojson](https://github.com/goccy/go-json)
 - [iterjson](https://github.com/json-iterator/go)
 
```
goos: darwin
goarch: amd64
BenchmarkUnMarshal-4              173359              6076 ns/op
BenchmarkGoJsonUnMarshal-4        732471              1552 ns/op
BenchmarkIterJsonUnMarshal-4      422719              2428 ns/op
BenchmarkJsonMarshal-4            486486              2239 ns/op
BenchmarkGoJsonMarshal-4         1000000              1019 ns/op
BenchmarkIterJsonMarshal-4        659385              1813 ns/op
PASS
ok      command-line-arguments  6.772s
```

# 总结

- 只需要部分值时候采取加载部分值，或序列化部分值的方法
- 使用代码生成方法
- 使用第三方库代替标准库
- 若序列化和反序列化并非性能瓶颈建议使用标准库
 