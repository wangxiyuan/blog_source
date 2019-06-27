---
title: Keystone Q版PTG总结(简译)
date: 2017-11-09 10:57:18
tags:
  - OpenStack
  - Keystone
categories: 翻译
---
> 原文是Keystone PTL Lance对Q版本PTG的总结，本文提取并翻译了其中的核心信息，请阅读原文获取更详细的信息：https://www.lbragstad.com/blog/keystone-queens-ptg-summary

# Policy in Code

[Policy in Code](https://governance.openstack.org/tc/goals/queens/policy-in-code.html)是指各OpenStack服务把默认的Policy规则整合到代码中，不再需要默认的Policy.json文件。目前Keystone、Nova、Cinder等项目已经完成了重构。但还有很多项目未完成，Keystone team在Q版会帮助那些未完成的项目实现Policy in Code功能。

# RBAC improvements

Keystone的RBAC中admin role的作用域一直是一个问题。在当前Keystone实现中，只要一个user在一个project中有admin权限，则这个user可以通过该权限访问其他project的资源。例如：默认Policy场景下，在Nova中，user A在project A中有admin role，在其他project中没有role。 但此时user A却可以通过``nova list --all-tenents``来获取所有project里的VM信息。
<!-- more -->
针对该问题，Keystone提出了global roles的概念，这是一种system-scope的role，即真正的全局admin。非global role的user将会被限制在具体的project里。因此，我们需要：

1. 新增system-scope概念。

2. 修改以前的policy规则。

   修改Policy会导致无法前向兼容，因此各服务在修改Policy前应该先新增废弃说明，过一两个版本后再删除。这也是Policy in Code的意义所在，Policy in Code把policy变成了类似配置项一样的东西，开发者可以对其进行配置、废弃等操作。在Policy in Code的基础上，给Policy新增scope属性，相当于给API限制了作用域，例如Keystone endpoint APIs是system scope的，而Nova的VM APIs则是project scope的。

# Application Credentials

怎么让一个外部应用或OpenStack服务使用Keystone鉴权服务？

比如Nova去Keystone做Token校验。目前OpenStack的方式是：1. 在Keystone中注册一个nova user。2. 在Nova配置项中记录nova user的name和password。

再比如一个外围应用想访问Nova。则该应用必须使用Keystone的user X。那么问题出现了：1.该应用有了user X的所有权限，然而管理员想让该应用只能访问Nova。2. user X的password暴露给了该应用，有安全风险。3. 如果修改了user X的密码，则该应用还要做对应的修改，否则无法使用。4. 如果user X被删除了，那么该应用直接无法使用。

针对以上问题，Keystone提出了新的使用方式：Application Credentials（AC）。AC有以下特点：

1. 不可改变。哪个user创建了AC，该AC就有这个user在创建时刻的role，并永久不变。
2. 每个user可以创建的AC有quota限制。
3. 创建AC的用户被删时，该AC也会被删除。
4. 可以对AC设置具体的API访问限制。
5. AC有过期时间。
6. AC可以被其他user（与创建user在同一project里的）删除。

更多细节，请参考[spec](http://specs.openstack.org/openstack/keystone-specs/specs/keystone/backlog/application-credentials.html)。

# Microversions

越来越多的OpenStack项目开始支持Microverions，Microversions的好处与坏处在这里不再细说。主要是为了解决API的兼容性。Keystone决定支持Microversions。具体的实现方式还未敲定，现在是信息收集阶段。

# Deprecations & Removals

1. V2 APIs将会被移除，计划在Q版移除90%，目前一些patch已经合入主干了。
2. UUID token已被废弃，JWT token正在开发中。
3. 由于UUID token的废弃，已经没有其他类型的Token是持久化存储的了，所有Token的SQL driver没有存在的必要了，会被废弃。
4. V2 APIs移除后，Catalog的SQL driver也没必要存在了。

# Mission Statement & Long-Term Vision

这个与OpenStack LTS类似，即做一个长时间支持的版本，更多的是为了商业化的考虑。OpenStack项目默认是支持两个版本，即N版本发布后，N-3的版本将会被移除。OpenStack LTS在本次悉尼的summit上也被热议。这里不再细说。

# Pike Retrospective

P版的回顾，不再赘述。
