---
title: Git代码管理
date: 2017-01-08 10:47:41
tags: [工具]
---

## 创建Git仓库

```

echo "# Git 工程" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/09jianfeng/AudioDemo.git
git push -u origin master

```

上面的代码指定了远程仓库为 `https://github.com/09jianfeng/AudioDemo.git `，并且把代码同步到了远程仓库

## 分支操作

### 创建分支

创建本地分支

```
git branch develope
```

创建服务器分支

```

```

### 删除本地分支

```
git branch -D develope
```

