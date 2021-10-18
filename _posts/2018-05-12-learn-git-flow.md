---
layout: post
title: 初学Git工作流程
date: 2018-05-12 16:23
tags: Git
categories: 编程
---

(文中图片均来源于网络)

## Git

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512b722a479~tplv-t2oaga2asx-image.image)

[Git](https://git-scm.com/)已是代码版本管理的标配，其分布式、多分支功能让人印象深刻。

## Git工作流程（Git Workflow）

![github_branching.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512b3dc3eeb~tplv-t2oaga2asx-image.image)


当项目需要多人共同开发时，规范工作流程就变得越来越重要。合适的工作流程能让多人协同开发更加顺利和高效。

目前主流的Git工作流程有三种：

- [Git Flow](http://nvie.com/posts/a-successful-git-branching-model/)（*版本发布*）
- [GitHub Flow](https://guides.github.com/introduction/flow/index.html)（*持续发布*）
- [GitLab Flow](https://docs.gitlab.com/ee/workflow/gitlab_flow.html)（*持续发布、版本发布*）

三种工作流程各有优缺点，对于不同类型的项目有各自的用武之地。笔者开发Android项目时使用的是Git Flow，对此比较熟悉。其余两种工作流程，笔者出于学习的目的，在文中谈谈自己的理解。

## Git Flow

从分支分类开始，有以下几类：

### 长期分支、主要分支

- **master**（主分支）：稳定可发布、产品线
- **develop**（开发分支）：处于开发状态

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512b4fef84c~tplv-t2oaga2asx-image.image)

```shell
git checkout -b develop master
git push origin develop
```

### 短期分支、支持性分支

- **feature**（功能分支）
- **release**（发布分支）
- **hotfix**（修复分支）

#### feature 功能分支

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512b4ded7a2~tplv-t2oaga2asx-image.image)

```shell
git checkout -b feature-main develop
# git commit 1
# git commit 2
# git commit 3
git checkout develop
git merge --no-ff feature-main
git branch -d feature-main
git push origin develop
```

##### --no-ff

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512b61146df~tplv-t2oaga2asx-image.image)

#### release 发布分支

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512b632efca~tplv-t2oaga2asx-image.image)

```shell
git checkout -b release-1.0 develop
# 可能在该阶段再分出 fix-* 分支来修复发布前的问题，会有git commit和 git merge 操作。
```

发布分支已测试完毕，问题已修复，可发布时，将代码同步到`master`分支中。

```shell
git checkout master
git merge --no-ff release-1.0
git tag -a v1.0
git push origin master
git push origin v1.0
```

若在发布分支中有修复问题，那么这些提交也要同步到`develop`分支中。

```shell
git checkout develop
git merge --no-ff release-1.0
git push origin develop
```

删除发布分支。

```shell
git branch -d release-1.0
```

#### hotfix 修复分支

当线上版本有紧急问题需要修复，`develop`分支还处于下一个版本的开发状态，不好从开发分支分出修复分支，选择从`master`分出`hotfix-*`分支来修复该紧急问题。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512e18b23d2~tplv-t2oaga2asx-image.image)

```shell
git checkout -b hotfix-1.2.1 master
git commit -a -m "Bumped version number to 1.2.1"
```

该紧急问题被修复，并验收通过时发布修复版本，同步代码到`master`分支。

```shell
git checkout master
git merge --no-ff hotfix-1.2.1
git tag -a v1.2.1
git push origin master
git push origin v1.2.1
```

接着将代码同步到`develop`分支。

```shell
git checkout develop
git merge --no-ff hotfix-1.2.1
git push origin develop
```

删除修复分支。

```shell
git branch -d hotfix-1.2.1
```

实际操作中，会将`master`作为开发分支，因为它几乎是Git相关工具的默认分支，可以省去大量切换工作。新建诸如`release`、`production`分支作为产品分支。

## GitHub Flow

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512e1b8ddcd~tplv-t2oaga2asx-image.image)

只有一个长期分支`master`，很适合持续发布的项目，如：网站，相比Git Flow 更简单、易用。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512e1ac5a87~tplv-t2oaga2asx-image.image)

```shell
git checkout -b bug47833 master
git commit
git checkout master
git merge --no-ff bug47833
git push origin master
```

## GitLab Flow

只有一个主分支`master`。该工作流程最大原则是“上游优先”，只有`master`分支的代码提交，才能应用到下游分支中。

在开发需求或修复问题时，可以使用GitHub Flow方式从`master`分支分出工作分支，开发完成后以合并请求合并到`master`分支，当验收通过时，就可以合入到下游分支并发布了。

### 持续发布

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512e1d50e6b~tplv-t2oaga2asx-image.image)


### 版本发布

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e512f6f76889~tplv-t2oaga2asx-image.image)

## 结束语

本文主要介绍了3种Git工作流程，其中重点介绍了Git Flow。目前笔者所在的团队使用的工作流程类似于GitLab Flow，在这里只是简单的介绍，而GitHub Flow工作流程未真正实践过，出现在文中是为了丰富文章内容。

文中的Git工作流程并不是全部，完全可以自己按需扩展或全新定义出适合项目的工作流程。

## 参考资料

- [Git 工作流程](http://www.ruanyifeng.com/blog/2015/12/git-workflow.html)
- [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow](http://scottchacon.com/2011/08/31/github-flow.html)
- [Introduction to GitLab Flow](https://docs.gitlab.com/ee/workflow/gitlab_flow.html)