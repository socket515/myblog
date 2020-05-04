---
title: viper改造
date: 2020-05-04 15:22:53
tags:
- golang
categories:
- golang
---
<meta name="referrer" content="no-referrer" />

## 背景
 线上某一项目使用viper作为配置库。其中使用到加载配置和动态加载远端配置功能。某一日突然发生了fatal error: concurrent map read and map write。原因在于viper对象是非线程安全的，项目上存在对viper对象读写没有上锁问题。

## 分析

### viper用法
 viper用于项目中配置管理。viper对象存储项目的默认配置和配置文件的配置，一般是在程序启动的时候设置好默认的配置和加载配置文件中的配置。在项目运行时候大多数情况是读操作，从viper中读取配置。viper还负责从远端加载动态配置，程序运行时候watch远端的更改定期跟新到viper中。而viper底层实现是由一个``kvstrore map[string]interface``存储配置信息，在运行会对``kvstrore``进行并发读写。

### 使用场景分析
 viper对象的``kvstore``是用于存储远端的kv数据。远端支持etcd和consul。``kvstore``是用于存储动态配置的。一般情况下配置的更改不会过于频繁,而读的操作会非常多，是一个读多写少的场景。

### 改进分析
 为保证viper对象线程安全,要对viper对象的读写操作都上锁。项目中多去使用到``viper.GetXX``,为viper对象读写操作在外部上锁会是一个庞大的工程。考虑到其他项目组使用的viper是否也存在这样的改造问题，这个改造工程会无比巨大。动态配置的读写是一个读非常多，写只有偶尔几次场景。为其增加读写锁可能会导致性能稍差，存在写饥饿的问题。改造viper库的实现从内部解决线程安全的问题。

## 改造
 viper对象负责管理程序中默认配置(由`defaults`存储)、配置文件配置(由`configs`存储)和动态配置(`kvstore`)。默认配置和配置文件配置的写入一般都在程序运行初始化的时候进行，不存在并发读写,这次改进针对`kvstore`改造。改造是基于1.5版本改造的。
 
 改造思路: 使用COW。写的时候复制原来的对象，写copy的对象，再将copy对象替换过去。使用`atomic.Value`存储对象的引用保证对引用的读写是原子性。

 使用atomic.Value替代原来的``map[string]interface{}``
 ```go
  ...
	config         map[string]interface{}
	override       map[string]interface{}
	defaults       map[string]interface{}
	kvatomic       atomic.Value
  ...
 ```
 
 初始化时候存储新建空的``map[string]interface{}``
 ```go
 	v.config = make(map[string]interface{})
 	v.override = make(map[string]interface{})
 	v.defaults = make(map[string]interface{})
 	v.kvatomic.Store(make(map[string]interface{}))
 ``` 
 
 读取时候从atomic.Value中原子性读取出引用
 ```go	
    // K/V store next
    kvstore := v.kvatomic.Load().(map[string]interface{})
    val = v.searchMap(kvstore, path)
    if val != nil {
    	return val
    }
    if nested && v.isPathShadowedInDeepMap(path, kvstore) != "" {
    	return nil
    }
 ```
 
 写时候先对比从远端读来的bytes和原来是否一致,不一致时再复制对象写入后再原子性写回去
 ```go
    // 判断bytes是否一致
 	hashKey := buf.String()
 	if hashKey != provider.HashKey() {
 		err = v.unmarshalReader(buf, kvStore)
 		provider.SetHashKey(hashKey)
 	}
 	
 	// 写时候先复制再写入
 	c := make(map[string]interface{})
 	for _, rp := range v.remoteProviders {
 		err := v.watchRemoteConfig(rp, c)
 		if err != nil {
 			continue
 		}
 		if len(c) > 0 {
 			// should over writer?
 			kvstore := v.getCopyKvStore()
 			for k, v := range c {
 				kvstore[k] = v}
 			v.kvatomic.Store(kvstore)
 		}
 		return nil
 	}
 	
 	// 复制函数
 	// getCopyKvStore get on copy viper.kvstore
    func (v *Viper) getCopyKvStore() map[string]interface{} {
    	val := v.kvatomic.Load().(map[string]interface{})
    	return deepCopy(val)
    }
 	
 	// deepCopy copy deps map
    func deepCopy(c map[string]interface{}) map[string]interface{} {
    	ret := make(map[string]interface{})
    	for key, val := range c {
    		switch val.(type) {
    		case map[interface{}]interface{}:
    			// nested map: cast and recursively deepCopy
    			val = cast.ToStringMap(val)
    			val = deepCopy(val.(map[string]interface{}))
    		case map[string]interface{}:
    			// nested map: recursively deepCopy
    			val = deepCopy(val.(map[string]interface{}))
    		}
    		ret[key] = val
    	}
    	return ret
    }
 ``` 
 
## 使用
 使用go modlue的项目可以轻易replace此项目
 
 ```go
 go mod edit --replace=github.com/spf13/viper@v1.5.0=github.com/Socketsj/viper@latest
 ```
 
 项目地址[https://github.com/Socketsj/viper](https://github.com/Socketsj/viper)

## 总结
 在解决线程安全问题的时候，并不是所有场景都使用锁。结合使用场景和性能问题找到最优解。
