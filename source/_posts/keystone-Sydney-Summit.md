---
title: Keystone Sydney Summit总结(简译)
date: 2017-11-16 11:13:18
tags:
  - OpenStack
  - Keystone
categories: 翻译
---
> 原文是Keystone PTL Lance Bragstad 参加OpenStack悉尼峰会对Keystone相关的信息的总结，本文提取其中的重点信息并翻译。请阅读原文获取更详细信息：https://www.lbragstad.com/blog/openstack-summit-sydney-recap

# Operator & User Feedback

主要是收集OpenStack用户和工程师对OpenStack的一些意见和看法。
<!-- more -->
## Application Credentials Feedback

之前PTG总结的[文章](http://www.wangxiyuan.top/2017/11/09/keystone-Q-PTG)中已经讲述了什么是Application Credentials（AC）。峰会中对该feature的设计没有什么改变。只是总结了几个核心观点：

1. 暂时不打算使用Policy（不是Role）来实现AC。
2. AC的生命周期与user强绑定，而不是project。
3. User不需要把password暴露给应用。
4. user也不需要为用户专门在Keysotne中创建服务账号。

目前未看到社区有对应的patch，估计Q版很难实现。

## RBAC & Policy Feedback

PTG总结中也提到了该问题。峰会上主要讨论了几个方面：

1. System role（也就时System-scoped token）。用户们对这个特性很感兴趣，Lance目前在负责该feature，代码和spec都编写完了，但review不足，刚兴趣的同学可以去review一下，加快该特性的合入。[Patch link](https://review.openstack.org/#/q/status:open+project:openstack/keystone+branch:master+topic:bp/system-scope)
2. 用户希望Keystone支持通过API动态修改或新增Policy的功能。该功能AWS已经支持了。

## Other Feedback

用户提了一个需求：在federation场景，IdP的用户信息不应该与OpenStack的用户有对应关系。其实Lance也不太理解该用户的需求。我也不班门弄斧了。感兴趣的同学可以详细看下Lance的原文。

# Recorded Talks

峰会中有几个涉及Keystone的topic，值得一看。

1. Colleen 讲述[Keystone Federation](https://www.openstack.org/videos/sydney-2017/demystifying-identity-federation)。干货满满的topic，可惜由于时间有限，只演示了Keystone as SP。而Keystone as IdP或Keystone to Keystone并有演示。
2. Lance讲述[Keystone RBAC](https://www.openstack.org/videos/sydney-2017/custom-rbac-can-i-do-that)。推荐看一下。
3. Lance讲述[Keystone Project Update](https://www.openstack.org/videos/sydney-2017/keystone-project-update)。

# JWT

AT&T对JWT很感兴趣，这意味着OpenStack与K8S可以使用一样格式的Token了，使得两个项目的交互便的更方便。后续的一个方向： 使用Keystone做K8S的鉴权系统。
