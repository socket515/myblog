---
title: hexo next 主题搭建个人博客
date: 2019-05-02 16:07:22
tags:
 - hexo
 - 博客
categories:
 - hexo
---
<meta name="referrer" content="no-referrer" />

这里提供一个博客模板，方便快速搭建起个人博客。模板地址[https://github.com/Socketsj/blog-template](https://github.com/Socketsj/blog-template)

搭建前需要了解git、node和hexo，这里不会详细展开，想要详细了解请看[hexo+github搭建个人博客](https://socketsj.github.io/2019/04/30/hexo1/)

# 前期准备

## 拉取项目

```
git clone https://github.com/Socketsj/blog-template.git
cd myblog
```

## 安装依赖

如果条件允许
```
npm i
```

一般无翻墙需要用到cnpm

```
cnpm install
```

# 配置个人信息
## site 配置
 打开根目录下的_config.yml，找到site
 
 ```
 # Site
 title: # 输入博客标题可以以你的姓名
 subtitle: 个人博客
 author: # 博客作者
 language: zh-Hans
 timezone:
 ```

## deploy
 同样是在根目录下_config.yml，找到Deployment
 
 ```
 # Deployment
 ## Docs: https://hexo.io/docs/deployment.html
 deploy:
   type: git
   repository: # 这里写你的username.github.io的仓库路径
   branch: master
 ```

## next主题
 可以根据喜好设置主题，在themes/next/_config.yml，找到Schemes。选择你想要的主题取消注释并将原来主题加上注释

 ```
 # Schemes
 #scheme: Muse
 #scheme: Mist
 scheme: Pisces
 #scheme: Gemini
 ```

## social
 个人社交配置在themes/next/_config.yml, 找到social,根据你要显示的社交信息取消注释填上对应信息

 ```
   social:
   #GitHub: https://github.com/yourusername || github
   #E-Mail: mailto:youmail@site.com || envelope
   #Weibo: https://weibo.com/yourname || weibo
   #Google: https://plus.google.com/yourname || google
   #Twitter: https://twitter.com/yourname || twitter
   #FB Page: https://www.facebook.com/yourname || facebook
   #VK Group: https://vk.com/yourname || vk
   #StackOverflow: https://stackoverflow.com/yourname || stack-overflow
   #YouTube: https://youtube.com/yourname || youtube
   #Instagram: https://instagram.com/yourname || instagram
   #Skype: skype:yourname?call|chat || skype
 ```

## 头像
  可以选择更换头像，将你要更换的图片放在themes/next/source/images/下，并命名为avatar.jpg

# 文章
文章在./source/_post/目录下的md文件, 文章以md形式编写，可以选择把该目录下的demo.md删掉重新写自己的文章

## 新建文章
执行新建文章命令会在./source/_post/目录下创建文章名称的.md文件

```
hexo new '文章名称'
```

## 编写文章

```
---
title: {{ title }} # 会自动生存文章名称
date: {{ date }} # 创建日期自动生成
tags: # 可以自行选择为文章打上标签
 - 例子 # 这样是打上例子标签
categories: # 写文章分类
 - 生活  # 这样是将文章分到生活类别
---
<meta name="referrer" content="no-referrer" />  # 用于加载网络图片所用不用管
......正文.....  # 此处编写正文
```

## 添加图片

图片分为网络图片和本地图片

### 网络图片

网络插入直接在文章照md格式写

```
![](http://xxxxxx/xxx.jpg)
```

### 本地图片

在./source/_post/创建文章名相同的名字

例如文章名字是example.md，那么同样在目录下创建example文件夹用作存放图片

图片资源存放在新建的文件夹中，插入图片如下写法

```
![](example/xxx.jpg)
```

# 调试&发布

## 本地调试

编写好文章好可以，在本地浏览器下查看运行效果

```
hexo s # 开启hexo server模式
```

浏览器输入[http:localhost:4000](http:localhost:4000)，即可查看到博客效果

## 发布文章

发布命令

```
hexo g # 编译静态文件
hexo d # 发布到githubpage
```
