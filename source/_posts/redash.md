---
title: redash
date: 2019-05-03 21:55:46
tags:
 - redash
 - python
categories:
 - python
---
<meta name="referrer" content="no-referrer" />

# redash 实现原理
![avatar](https://raw.githubusercontent.com/Socketsj/redash/master/images/head.jpg)


* [前言](#前言)
* [Redash简介](#Redash简介)
* [相关技术](#相关技术)
* [技术架构](#技术架构)
* [一次查询过程](#一次查询过程)
* [代码结构](#代码结构)
* [模型关系](#模型关系)


## 前言
 前一段时间拜读过redash源码和部署过redash，经过几番周折，总算对redash有一定认识。在此记录前段时间的学习成果，此篇旨在抛砖引玉，如有错误请指出。

## Redash简介
 这里啰嗦一下，可能有一些大佬们不了解redash。redash是一款融合多数据源的可视化查询工具，可以说一个支持多数据源的数据可视化工具。深入了解前往[redash文档](https://redash.io/help/)

## 相关技术
### angular
  前端使用angular版本1.5.8，对于当前版本算是比较旧的版本。这可能是redash项目开发较早和前端技术快速迭代更新有关。
### python
  后端使用python2.7编写，算上一个"历久弥新"的版本
### flask
  使用flask作为web框架，处理http请求，[flask文档](https://dormousehole.readthedocs.io/en/latest/)
### sqlalchemy
  作为Redash的orm，用于redash后台信息的增删查改，如记录数据源，记录查询query等,[sqlalchemy](https://docs.sqlalchemy.org/en/13/)
### celery
  celery是异步任务队列，用于redash中的异步任务，要理解redash原理必须先了解[celery](http://docs.jinkan.org/docs/celery/)

## 技术架构

![avatar](https://raw.githubusercontent.com/Socketsj/redash/master/images/architecture.jpg)

  Redash采用的Celery异步架构，大部分功能模块都由异步任务完成。
  
  redis作为消息中间件，每个执行异步任务的worker之间都是独立的。
  
  query_runner是支持多数据源查询的关键部分，提供了各种数据源的查询引擎。
  
  Postgresql存储查询过程中产生的数据以及配置信息如数据源。
  
  webserve负责接受前端发来的请求发起异步任务。
  
  前端主要是获取到后端返回数据，控制页面元素和css样式，提供可视化组件和基本功能。
所有与后端交互部分js逻辑处理，实现数据获取与功能跳转。

## 一次查询过程
 查询功能是Redash的核心功能，通过了解查询过程结合架构图能更清晰了解Redash工作原理

 编写好sql点击execute共有以下操作
 
 - 发送查询query和data_source_id到后端
 - 后端检查相关权限后，发起查询异步任务并返回job_id
 - worker接受任务后，调用query_runner查询数据，查询完成后将结果查询到存到query_result，redis记录result_id
 - 前端发起events请求，后端接口接受并记录这次操作
 - 前端轮询job_id,后端判断这次job是否完成，若完成返回query_result_id
 - 前端发送query_result_id，后端接收后查询对应query_result返回，查询结束

## 代码结构
 Redash代码结构模块化做的非常好，通过目录名可以分析出代码功能定位


![avatar](https://raw.githubusercontent.com/Socketsj/redash/master/images/code.png)

 - client 此目录下主要是前端文件如html模板、js文件和css文件等
 - redash 此目录为后端逻辑也是Redash的核心代码部分
 - test 针对redash目录中代码单元测试代码

 以下是Redash目录下的结构

![avatar](https://raw.githubusercontent.com/Socketsj/redash/master/images/code2.png)
 
 - authentication 权限认证模块如权限认证和登录相关功能
 - cli Redash的命令行操作如建表、添加数据源、测试函数等。提供命令行操作接口
 - destination Redash上通知模块，定义通知类型如邮件通知
 - handles 后端接口定义
 - metrics 统计指标相关
 - models 模型定义，后台模型类定义是Redash设计核心
 - query_runner Redash的查询引擎，支持各种数据源关键部分
 - settings 配置文件
 - tasks 定时任务和异步任务定义
 - templates html模板如发送邮件模板等
 - util 通用工具包

## 模型关系
 模型关系是Redash后台设计的核心部分，模型定义了Redash后台功能。
 由于model过多，为方便了解模型一些关系连线会删除

![avatar](https://raw.githubusercontent.com/Socketsj/redash/master/images/model.png)
 
 - access_permission, 权限表
 - alembic_version,版本号
 - alert_subscriptions，报警描述
 - alerts，报警列表
 - api_keys, api key管理
 - changes，升级改动
 - dashboards，报表存储
 - data_source_groups,每个组对应的数据源
 - data_sources，所有数据源
 - events，后台日志
 - groups，所有分组列表
 - notification_destinations，报警的模板和目的地
 - organizations， 组织
 - queries，所有queris
 - query_results，query的结果，另一种缓存
 - query_snippets，SQL的评论
 - users, 用户已列表
 - visualizations，可视化图表存储
 - widgets，可视化控件