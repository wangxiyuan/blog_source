---
title: Keystone Token详解(WIP)
date: 2017-11-08 11:48:49
tags:
  - OpenStack
  - Keystone
categories: OpenStack
---
# 概念

Keystone作为OpenStack的大门，起着路标（endpoint）、身份校验（credential）、权限控制（role）等作用。对于校验通过的用户，Keystone对其颁发Token。用户拿着这把Token钥匙，就可以访问其他OpenStack的服务了。
<!-- more -->
## Token的类型

到目前Q版为止，Keystone前后一共支持以下几种Token类型：

1. UUID (已废弃，计划R版本删除)。

   这种Token是最简单的入门级Token，是一串在OpenStack中随处可以看到的简单的UUID。我们可以在python环境中简单的生成一个类似的：

   ```python
   >>> import uuid
   >>> uuid.uuid4()
   UUID('0e1d49a9-6141-4f5b-af66-a93c373d9369')
   >>> uuid.uuid4().hex
   '85697af3031d42c4a1b2e6165dea0ac7'
   ```

   因此，UUID Token本身不包含任何用户的信息。例如：当用户使用该Token访问Nova时，Nova首先要拿该Token去Keystone校验，Keystone在拿到该Token后，去存储后端（目前只支持SQL）中获取该Token的信息，并同时读取相关用户的信息，拼装后返回给Nova。流程如下：

   ![uuid](/images/uuid-token-flow.png)

   之前在[Keystone Cache](http://www.wangxiyuan.top/2017/10/19/keystone-cache/)这篇文章中提到过，在并发场景中，REST请求和DB query往往是是性能瓶颈所在。因此UUID Token的校验是一个耗时的操作。所以社区把它废弃了。

2. PKI/PIKZ Token（已移除）。

   TBD

3. Fernet Token（默认的Token）。

   TBD

4. Json Web Token（正在开发中）。

   TBD

## Token的获取

TBD

## Token的校验



## Token的接口

TBD

# Token认证流程

TBD

# 总结

TBD
