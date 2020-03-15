---
title: 一次排查etcd mvcc database space exceeded的问题
date: 2020-03-15 11:02:34
tags:
- golang
- go-micro
- etcd
categories:
- 记录排查
---
<meta name="referrer" content="no-referrer" />

## 前言
 一个宁静的夜晚里，群里报警机器人不断发出错误报警。查询错误日志``Error: rpc error: code = 8 desc = etcdserver: mvcc:database space exceeded``. 发现是etcd出现异常，针对这一次异常进行排查。

## 背景
 项目使用微服务架构，编程语言golang。使用go-micro作为微服务框架。etcd用作服务注册中心，用于服务注册和服务发现。etcd出现异常则整个服务处于不可用状态。

## 原因分析
 从错误信息中得知``mvcc: database space exceeded``一般为etcd内存达到上限。etcd使用内存限制为2GB, 当内存达到上限导致数据无法写入。etcd默认不会compact。频繁地变更和写入数据会导致历史数据堆积，超过内存无法写入。需要开启``auto-compact``

 修改etcd启动参数的方法
 ```
 ExecStart=/usr/local/bin/etcd \
 -name etcd1 \
 -data-dir /var/lib/etcd \
 --advertise-client-urls http://127.0.0.1:2379 \
 --listen-client-urls http://127.0.0.1:2379 \
 --auto-compaction-retention=1 \     #开启每隔一个小时自动压缩
 ```
 
## 深入分析
 etcd只是用于服务注册和服务发现，应该不会出现频繁写入导致历史数据堆积。若只是用于服务发现、服务注册以及配置中心，不应该出现内存写满，压缩数据只是治标不治本。由此阅读go-micro的服务注册相关代码。
 
 加入log分析每次服务注册内容
 ```go
    // register 代码片段
	service := &registry.Service{
		Name:      config.Name,
		Version:   config.Version,
		Nodes:     []*registry.Node{node},
		Endpoints: endpoints,
	}

	g.Lock()
	registered := g.registered
	g.Unlock()

	if !registered {
		log.Logf("Registering node: %s", node.Id)
	}

	// create registry options
	rOpts := []registry.RegisterOption{registry.RegisterTTL(config.RegisterTTL)}

    log.Info("service, ", service) // 打印log查看每次注册内容
	if err := config.Registry.Register(service, rOpts...); err != nil {
		return err
	}
 ```
 
 查看log内容后有惊人发现，每次log内容都有可能不一样。service里的subcribers的顺序有时候A在先，有时候B在先。
 
 ``` json
 {
  "name": "subcriberA"
  "request": xxxx,
  "response": xxxx,
 }, 
 {
   "name": "subcriberB"
   "request": xxxx,
   "response": xxxx,
  }
  
  
 {
    "name": "subcriberB"
    "request": xxxx,
    "response": xxxx,
 }, 
 {
     "name": "subcriberA"
     "request": xxxx,
     "response": xxxx,
 }
 ```
 
 查看subcribers的排序代码
 
 ```go
	var subscriberList []*subscriber
	for e := range g.subscribers {
		// Only advertise non internal subscribers
		if !e.Options().Internal {
			subscriberList = append(subscriberList, e)
		}
	}
	sort.Slice(subscriberList, func(i, j int) bool {
		return subscriberList[i].topic > subscriberList[j].topic
	})
 ```
 
 subscriberList以topic作为key排序。A和B刚好关注了同一个topic。由于sort使用的是三路快排，是一个不稳定的排序方法。导致subscriberList顺序不一样，每次注册写入etcd的值不一样导致了历史数据堆积。

## 总结
 业务逻辑需要两个subscriber需要订阅同一个topic监听相关变动。因为subscriberList以topic作为key排序,所以一个服务中subscriber不能关注同一个topic，否则会导致服务注册时把注册中心写满。在当前最新版本2.2.0中依然是使用topic作为排序key，我认为不应该以topic作为排序key应该以name作为排序key，这样导致业务代码必须避免关注同一个topic问题。
 