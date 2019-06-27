---
title: Keystone入门到精通：1.概述
date: 2018-04-21 04:10:12
tags:
  - OpenStack
  - Keystone
categories: OpenStack
---
# 前言

2018年3月，OpenStack社区发布了其第17个版本，代号Queen。不知不觉，OpenStack已经走过了9个年头。不可否认，OpenStack的热度以及开发者数量在逐步降低与减少。但这恰恰能反映出OpenStack的越来越成熟以及核心组件的越来越稳定。任何产品随着版本的不断迭代、功能的不断完善，势必会进去一个平稳期，此时产品的关注点聚焦于稳定性、易用性等问题上。OpenStack亦然。这时，我们再打开来看OpenStack，不但能更深入的理解其原理与功能，还能降低因代码快速迭代而导致的知识过时、跟不上社区步骤的问题。现在在网上搜到的OpenStack文章，其大多内容与当前的OpenStack相差甚远，很多提及的命令、配置甚至架构已不再适用当前的版本。为了帮助OpenStack开发者、用户、管理员快速上手OpenStack，在保证真正与实时Upstream一致的前提下，我计划从用户以及开发者两个角度，以blog的方式，逐步介绍Keystone以及一些其他周边项目的原理与实现。希望能帮助国内更多的开发者了解OpenStack，甚至进一步参与OpenStack的开源开发，为开源云生态添砖加瓦。

<!-- more -->

# 什么是Keystone

Keystone是OpenStack基础项目之一，是所有OpenStack服务的入口应用，提供用户管理，鉴权授权、服务发现以及配额管理的功能。在讲解Keystone功能之前，读者需要先了解Keystone中的一些基本概念。其他的高级概念在后续文章中逐步提出，本文作为入门章节，先不做讲述。

## 基本概念

**User**（用户）: 云场景下最小粒度的访问资源对象，一个user就是一个访问云系统的个体。如在AWS中注册一个账号，该账号就是AWS中的一个用户。

**Project**（租户）：租户用来实现资源隔离，不同租户中的资源相互不可见。

**Domain**（域）： 与Project类似，但比Project更高一级，用来隔离租户、资源。一个Domain中可以包含多个Project。

**Role**（角色）：不同的角色代表了不同的权限。表示User与Project/Domain之间的关系。一个User在一个Project中有什么的Role，决定了该用户在该Project中有什么权限。User与Project通过Role进行绑定，也可以与Domain通过Role绑定。

**Service**（服务）：OpenStack对外提供的如计算、网络、存储等服务需要先注册在Keystone中，对应Keystone的service概念。

**Endpoint**（端点）：每个在Keystone中组成的服务都有唯一的应用入口URL，即就是Endpoint。

**Region**（区）：逻辑上的地理概念，一套OpenStack可能同时管理北京、上海两地的物理设备，北京、上海分别对外提供计算、网络、存储服务，那么就可以把北京、上海划分为不同的Region，实现服务之间的隔离。

**Token**（令牌）：一个Token中包含了一个用户的有效认证信息，用户必须携带Token访问各个OpenStack服务。

## 基本功能

**用户管理**：Keystone提供了一组API，用来创建、修改、查询、删除用户。每个用户有唯一的密码。

**鉴权**：用户可以通过用户名/密码的方式（还有其他方式，本文暂不涉及）访问Keystone进行鉴权，鉴权通过后，Keystone向用户发放Token。用户可以使用该Token访问其他OpenStack服务。

**授权**：不同用户在同一租户/域中有不同的角色、同一用户在不同租户/域中也可以有不同的角色，角色决定一个用户的权限。一个授权的过程就是给一个用户在一个租户/域中赋予相应角色的过程。

**服务发现**： 在用户鉴权通过后，Keystone返回给用户的Token中包含了在当前Keystone中已注册的服务的访问地址。用户可以使用Token中的服务的URL访问相应服务。

## 基本流程

以一个用户创建一台虚拟机为例，我们来看下整个调用链：

![uuid](/images/keystone-start-1.png)

通过这个流程，以小见大，我们可以总结几点，它们适用于整个OpenStack：

1. Keystone是请求的入口，在一个灵活的OpenStack中，由于有服务发现功能，用户无需知道所有服务的地址。
2. Token作为鉴权、授权、用户信息、服务发现的载体，贯穿整个流程。
3. Keystone集中管理鉴权功能。
4. Keystone管理授权功能，但权限的校验却在各个服务中。

# 总结

通过本文，读者应该对Keystone有了一个宏观上的了解，在明白了“是什么”的基础上，后续我们会对Keystone的每个概念、每个功能逐步展开，让读者真正的理解、掌握Keystone。下一篇将会介绍Keystone的安装、启动、以及初始化过程。感兴趣的同学请多多关注。

