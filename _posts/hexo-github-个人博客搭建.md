---
title: hexo github 个人博客搭建
date: 2016-03-17 17:29:51
tags: [工具]
---

## 本地配置环境
 1、下载nodejs [下载链接](https://nodejs.org/en/),选择一个稳定的版本安装
 
 2、[hexo](https://hexo.io/) 官网，去官网看hexo的安装
 
 3、执行命令，在本地开启nodejs服务器
 
 ```
 hexo s
 ```
在浏览器上输入 0.0.0.0：4000 可以看到blog的样子

---
## 发布到github
### github创建工程
* 登录你的github
* 创建一个工程，名为 `你的用户名.github.io`


### 修改blog下的配置文件
编辑配置文件 _config.yml

```
deploy:
  type: git
  repository: git@github.com:你的帐号/你的帐号.github.com.git
  例如我的：repository: git@github.com:zhongbaitu/zhongbaitu.github.com.git
  branch: master
```
然后执行命令：

```
npm install hexo-deployer-git --save
hexo clean
hexo generate
hexo deploy
```
上述命令中：

* 执行 hexo generate会在本地生成一个public文件夹，里面是静态文件
* 执行 hexo deploy 把生成的静态文件push到你的github

去浏览器输入地址  ：  `你的用户名.github.io`，可以看到你创建的hexo blog已经上传到线上环境了

 ---
## hexo命令生成博客
#### 生成新blog

```
hexo new "newBlog"
```
上述命令会生成一个 `newBlog.md`文件在 source/_posts/ 文件夹下面。 用`mardown`语法编写你要发布的文档。编写好即可以。  运行 hexo s 开启服务，0.0.0.0:4000 打开可以先预览一下写的文章。

#### push到github
先生成

```
hexo g
```

push到github

```
hexo d
```

发布成功


## 主题
NexT主题 <http://theme-next.iissnan.com/getting-started.html>

### 标签
```
hexo new page "tags"
```

Then, add this line to source/tags/index.md that was created

```
type: "tags"
```


跟多信息看 next的 [主题wiki](https://github.com/iissnan/hexo-theme-next/wiki/)



