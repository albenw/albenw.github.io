title: Code Review方案
author: alben.wong
abbrlink: 4f7bd265
tags:
  - code review
categories:
  - routine
keywords: code review
description: >-
  Code Review是软件工程中重要的一环，特别是在大公司规范的流程中。大家也逐渐意识到Code
  Review的重要性以及它带来的好处，本文跟大家讨论一下Code Review的实践方案。
date: 2019-04-11 23:14:00
---
## 概要
Code Review是软件工程中重要的一环，特别是在大公司规范的流程中。大家也逐渐意识到Code Review的重要性以及它带来的好处，本文跟大家讨论一下Code Review的实践方案。

## 作用
Code Review的好处有很多，巴拉巴拉可以说一堆，但总的来说有以下几点：
- 找bug
- 提高代码质量
- 相互学习

## 实践方案
Code Review并没有一个标准应该怎么去做，每个公司甚至每个团队可能都有自己的做法，下面我讲3种暂时看到的做法
- 用 gerrit 
之前用过一下，觉得比较麻烦，主要是流程上体验不好，commit 之后要CR通过才能真正的提交到 gitlab，如果进度比较赶或者多人协作，别人需要用到你的代码，这就会带来时间上的拖延

- Commit + Issue + Label 
1）添加Project Label：Review Done；
2）设置做Code Review的默认分支Default branch，在Project Setting里面修改；
3）通知团队，新任务，使用Issue登记内容，代码Commit时，填写“#Num”关联Issue；
4）做完Code Review后，在Issue打上标签Review Done，并close issue，完成整个任务的流程；

- Merge Request + Label
在mr里面，reivew完代码直接打上标签；

## 总结
我个人推荐方案2 
我觉得主要区别在CR的时间点上，gerrit 在每一次的提交，这个比较强制了，不够灵活；方案3要在合并到别的分支时；方案2则比较灵活。
具体更详细的实践可以参考[基于 GitLab 的简单项目管理与协作流程](https://www.zlovezl.cn/articles/project-manage-with-gitlab/)