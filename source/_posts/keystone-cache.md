---
title: Keystone的cache机制
date: 2017-10-19 18:58:59
tags:
  - OpenStack
  - Keystone
categories: OpenStack
---

Keystone作为OpenStack的入口服务，它的性能极大地影响着整套OpenStack环境的体验。而在用户使用过程中，与Keystone打交道最多的地方在于获取token，以及校验token。并且往往伴随着高并发的问题。

如何提高Keystone的性能成为目前Keystone开发者最关注的问题之一。那么，在“申请Token“与”校验Token“这两个最常用、最重要的流程中，我们该如何优化？其实可以做的事情有很多，包括：部署Haproxy来分流API请求、使用非持久化存储的Fernet类型token来减少DB交互、使用缓存机制避免多次API请求或者程序运算等等。首先我们先从Keystone的缓存机制开始讲起。
<!-- more -->
# 机制说明

## Keystonemiddleware的缓存机制

Keystonemiddleware被集成在各个服务中。各服务通过Keystonemiddleware与Keystone交互，对用户携带的Token进行有效性校验。在没有缓存的场景下，服务每接收到一个Token，都会通过Keystonemiddleware访问Keystone（PKI Token除外）校验Token。因此在多服务并行、用户请求频繁的场景下，Keystone会接受海量的Token校验请求，成为整个流程的性能瓶颈所在。Keystonemiddleware中的Token的缓存机制可以有效的解决该问题，即把已校验的Token缓存在各服务本地，当服务再次接收到相同的Token时，不再需要去Keystone校验，直接在本地缓存中校验即可。这样就减少了与Keystone交互的次数，提高了性能。

那么这些数据具体缓存在哪里？Keystonemiddleware提供了两种缓存后端：
1. 缓存在服务本身的内容中。（以下简称IC（Inside Cache））
Keystonemiddleware默认采用这种方法，所有被服务校验过的Token都会被缓存到本服务内存中，当服务再次接收到相同的Token时，先在本服务内存中查找是否包含该Token，如果有，则直接在本地校验。Token的默认缓存时长是600秒（可配置），即每个Token会在内存中缓存5分钟。
2. 缓存在第三方存储后端中，如被广泛使用的memcached。（以下简称OC（Outside Cache））
校验原理与第一种类似，只是存储的位置不同，不再放在每个服务内。而是使用外围的缓存系统。

需要注意的是，Keytonemiddleware的缓存机制同样引出了一些其他的问题：如，缓存数据与Keystone本身存储的数据不一致的问题。假如一个有效Token被服务成功缓存后，管理员在Keystone中对该Token进行了Revoke操作（删除该Token，使之失效），或者修改了该Token的权限（如把Role从admin改到member），而这个操作各服务并不感知。因此当用户带着这个已经失效/降级的Token再次访问服务时，服务在Cache中校验通过，认为依旧有效/有权限。这就产生了Token不一致并且不可接受的结果。

那么如何避免这个问题? 如果使用IC方式的话，该问题无法避免，因为缓存数据存储在各服务的运行内存的堆栈中，其他进程无法直接修改。可能的解决方法有两种：
1. 各服务暴露修改Cache的API。
不可行：代价太大，并且需要Keysonte频繁主动地调用，严重影响性能。
2. 使用Hack方法，跨进程访问内存。
不可行：违反程序设计原理，并且在容器化场景下（资源隔离）无法使用。

因此，不建议在生产环境中使用IC方法（PKI Token除外，因为在PKI场景下，Keystonemiddleware会主动的周期性访问Keystone，刷新IC）。并且OpenStack社区在一年前也废弃了IC。

而使用OC方式，则可以避免该问题，具体方法在下一节（Keystone本身的缓存机制）中讲述。

## Keystone本身的缓存机制

Keystone本身的缓存对象有很多，不仅仅限于Token，还包括Catalog、Assignments(Role)、Revoke、Id_mapping(Federation)、Domain_config和Resource（Domain and Project）等。Keystone使用Oslo.cache库实现缓存功能（其他有Cache功能的服务基本也是这样）。而Oslo.cache本身使用Dogpile.cache库实现。因此，OpenStack中各服务的缓存能力要依靠Dogpile.cache。Dogpile.cache是一个python库，以plugin的方式向下支持多种缓存后端，向上提供统一的缓存API。目前支持的后端有：
- dogpile.cache.memcached
使用python-memcached库的memcached后端。
- dogpile.cache.pylibmc
使用pylibmc库的memcached后端。
- dogpile.cache.bmemcached
使用python-binary-memcached库的memcached后端。
- dogpile.cache.redis
Redis后端。
- dogpile.cache.dbm
本地DBM文件后端。
- dogpile.cache.memory
与Keystonemiddleware IC一样的内存后端。
- oslo_cache.mongo
MongoDB后端。
- oslo_cache.memcache_pool
memcached后端，但经过oslo_cache优化，建议连接数大于100时使用。
- oslo_cahce.dict
与Keystonemiddleware IC一样的内存后端，但使用字典存储。
- oslo_cache.etcd3gw
etcd3.X后端。

关系图如下所示：
```flow
st=>start: Cache backend
op1=>operation: Dogpile.cache
op2=>operation: Oslo.cache
e=>end: Keystone

st(right)->op1(right)->op2(right)->e

```

以Token为例，Keystone会把认证过的有效的Token缓存到Cache当中，当下次同样的Token进来时，直接从缓存中读取，提高了Token校验的性能。当Token被revoke或者Token信息变动时，Keystone会实时刷新缓存数据，使之与实际数据保持一致。基于此，当使用非memory（IC）的后端时，Keystone与各服务缓存数据不一致的问题就可以通过以下方法有效的解决：
1. Keystone和各服务使用同一缓存系统。
这样的架构下，Keystone来保证缓存系统中数据的一致性。各服务从keystonemiddleware拿到相同的缓存数据。
2. Keystone和各服务使用不同的缓存系统，使用外围工具保证各缓存系统之间数据的同步。
这样的架构下，外围工具周期性同步缓存数据，保证数据的一致性。但会有一定的延时。

# 如何使用

## Keystonemiddleware的配置与使用

Keytonmiddleware默认就已经开启了cache功能（无法关闭），并且默认使用IC。因此在生产环境中，想要使用OC，则至少需要配置``memcached_servers``配置项。
目前相关的配置项如下所示：

    注意：Swift与Keystonemiddleware集成时，配置方法较为特殊，本文暂不涉及。

| 配置项                         | 默认值   | 备注                                                |
| ------------------------------ | -------- | --------------------------------------------------- |
| memcached_servers              | None     | OC链接url，e.g.localhost:11211                      |
| token_cache_time               | 300（s） | Token缓存时长                                       |
| memcache_security_strategy     | None     | Token缓存数据的加密策略。可选项：None、MAC、ENCRYPT |
| memcache_secret_key            | None     | Token缓存数据的加密Key                              |
| memcache_pool_maxsize          | 10       | 可连接的socket的最大个数                            |
| memcache_pool_dead_retry       | 300（s） | Socket中断后，重连的等待时长                        |
| memcache_pool_socket_timeout   | 3（s）   | Socket超时时间                                      |
| memcache_pool_unused_timeout   | 60（s）  | 关闭未使用的Socket前的等待时长                      |
| memcache_pool_conn_get_timeout | 10（s）  | Socket连接的客户端等待时长                          |
| memcache_use_advanced_pool     | False    | 是否使用advanced的方式，只在python2.X中适用         |

## Keystone的配置与使用

上面讲到Keystone支持多种资源的缓存，因此Keystone同时也支持分别开启或关闭各个资源的缓存。默认情况下，各个资源的缓存开关已打开，但总缓存开关未打开。即Keystone默认未开启缓存功能。配置项如下：
> [cache]
> enabled = false
> [token]
> caching = true
> [revoke]
> caching = true
> ...

因此只需要配置[cache]里面的值即可。例如：
>   [cache]
>   enabled = true
>   backend = dogpile.cache.memcached
>   backend_argument = url:127.0.0.1:11211

所有相关配置项如下。

| 配置项                               | 默认值             | 备注                                                |
| ------------------------------------ | ------------------ | --------------------------------------------------- |
| config_prefix                        | cache.oslo         | backend的前缀，一般不需要修改                       |
| expiration_time                      | 600（s）           | 资源的缓存时长                                      |
| backend                              | dogpile.cache.null | backend的类型，在上文中已有详细说明                 |
| backend_argument                     | []                 | 缓存后端需要用到的参数，用：分隔                    |
| proxies                              | []                 | dogpile的proxy机制，暂不细说                        |
| enabled                              | False              | 是否开启缓存功能                                    |
| debug_cache_backend                  | False              | 是否开启debug日志                                   |
| memcache_servers                     | 60（s）            | 关闭未使用的Socket前的等待时长                      |
| memcache_pool_maxsize                | 10                 | 可连接的socket的最大个数                            |
| memcache_dead_retry                  | 300（s）           | Socket中断后，重连的等待时长                        |
| memcache_socket_timeout              | 3（s）             | Socket超时时间                                      |
| memcache_pool_unused_timeout         | 60（s）            | 关闭未使用的Socket前的等待时长                      |
| memcache_pool_connection_get_timeout | 10（s）            | Socket连接的客户端等待时长                          |

# 总结

从减少API交互次数的角度来看，在Keystone和各服务中使用cache功能可以很大程度的提高OpenStack整体的性能，但什么使用场景使用什么后端？这还需要用户自己不断摸索与实践。最后送给大家一个官方的推荐：
> For eventlet-based or environments with hundreds of threaded servers, Memcache with pooling (oslo_cache.memcache_pool) is recommended. For environments with less than 100 threaded servers, Memcached (dogpile.cache.memcached) or Redis (dogpile.cache.redis) is recommended. Test environments with a single instance of the server can use the dogpile.cache.memory backend.
