---
title: Keystone Trust机制
date: 2017-11-07 10:22:53
tags:
  - OpenStack
  - Keystone
categories: OpenStack
---
# 概念

Keystone的trust提供级联授权、代理授权的功能。在讲原理之前，先理解一下几个重要的基本概念：
<!-- more -->
1. **Trustor**:：该用户可以把自己的权限授权给其他用户。

2. **Trustee**： 被赋予其他用户权限的用户。

3. **User ID**：Trustor和Trustee的ID。

4. **Privileges**：Trustor赋予TTrustee的具体权限内容。例如：用户A在租户M中有角色X、Y、Z，现在用户A想把他在租户M中的角色X赋予用户B，则该动作就是具体的Privileges。

   > 注意：当使用Trust赋予用户权限时，必须指定Privileges。不指定的话，默认不赋予任何权限。

5. **Delegation depth**：Trust可传递的次数。即Trustee是否可以再次把Trustor的权限赋予其他用户。例如：用户A把他在租户M中的角色X赋予用户B，此时用户B可以继续把该角色（租户M中的角色X）赋予用户C。取值范围：

   - ``0``：表示权限不可以传递。
   - ``1``：表示权限可以传递，但只能传递一次，即A赋予权限给B后，B可以传递给C，但C不再能继续传递给其他用户。
   - ``X``：表示权限可以传递，只能传递X次。
   - ``inf``：表示权限可以无限传递。

6. **Duration**：指一次Trust的作用时间范围，包括起止时间。创建Trust时一般指定过期时间即可。

# 使用

Keystone提供的相关接口：

|   CRUD   |                   URL                    |           Command            |
| :------: | :--------------------------------------: | :--------------------------: |
|   GET    |             /OS-TRUST/trusts             |     openstack trust list     |
|   POST   |             /OS-TRUST/trusts             |    openstack trust create    |
| GET/HEAD |       /OS-TRUST/trusts/{trust_id}        |  openstack trust show *xxx*  |
|  DELETE  |       /OS-TRUST/trusts/{trust_id}        | openstack trust delete *xxx* |
| GET/HEAD |    /OS-TRUST/trusts/{trust_id}/roles     |            *None*            |
| GET/HEAD | /OS-TRUST/trusts/{trust_id}/roles/{role_id} |            *None*            |

1. 创建一个Trust。

   CLI：

   ```shell
   $ openstack trust create --project <project> --role <role> <trustor-user> <trustee-user> [--impersonate] [--expiration <expiration>] [--project-domain <project-domain>] [--trustor-domain <trustor-domain>] [--trustee-domain <trustee-domain>]
   ```

   Curl：

   ```http
   POST /v3/OS-TRUST/trusts
   Body:
   {
     'trust':{
       'trustor_user_id': '',
       'trustee_user_id': '',
       'impersonation': True/False,
       'project_id': '',
       'remaining_uses': int<minimum:1>,
       'expires_at': date format,
       'allow_redelegation': True/False,
       'redelegation_count': int<minimum:0>
       'roles': [{'id': 'xxx'}, {'name': 'xxx'}...]    
     }
   }
   ```

   需要特别说明的几点：

   - 该请求必须由Trustor发起。
   - ``--project`` 与``--role``需成对出现。即指定了Project时，必须也指定对应的Role。
   - ``--impersonate``也是必选参数，openstack client中默认为False，所以使用CLI时可以不带，但使用curl时必须指定。当``impersonate=True``时，表示trustee用该trust申请的token中的user信息是trustor，而不是本身。
   - ``remaining_uses``表示该trust还能再使用几次（即使用该trust获取token的次数）。默认是null，表示可以无限使用。与``allow_redelegation``不能共存。
   - ``allow_redelegation``表示是否允许trust传递。
   - ``redelegation_count``表示trust允许传递的次数，每传递一次，该值在trustor的基础上默认-1。

2. 获取一个Trust Scope的Token。

   当trust创建好后，可以使用该trust为trustee生成对应的token。

   CLI：

   ```shell
   $ openstack token issue --trust-id xxxx
   ```

   Curl:

   ```http
   POST /v3/auth/token
   Body:
   {
       "auth": {
           "identity": {
               "methods": [
                   "password"
               ],
               "password": {
                   "user": {
                         "domain": {
                           "name": "default"
                       },
                       "name": "admin",
                       "password": "root"
                   }
               }
           },
           "scope": {
               "OS-TRUST:trust": {
                   "id": xxxx
               }
           }
       }
   }
   ```



# 原理

Trust原理很简单，就是通过trustee集成trustor的role，来获取对应的权限。这里讲几个需要注意的点：

1. Trust的创建。

   1. Policy要求：只有trustor才能创建trust。

      > ```
      > 'user_id:%(trust.trustor_user_id)s'
      > ```

   2. 如果创建trust的用户A本身就是一个trustee（使用trust scoped token），则``--roles``与用户A的trustor的``--project``有对应关系即可(role assignment)。

   3. Keystone提供一个配置项：``max_redelegation_count``表示trust允许的最大传递次数。

   4. Trust有个传递链，因此每次创建Trust时，Keystone都要依次检查传递链，保证该trust链中每个trust的``redelegation_count``、``expires_at``、``role``、``impersonation``等属性都要保持行为正常。

2. Trust Scoped Token的获取。

   1. Keystone根据request body中的scope来决定token的作用域，Trust的scope关键字是``OS-TRUST:trust``，此外还支持``project``、``domain``以及不带Scope的Unscope方式。
   2. 根据trust的``impersonation``属性，Keystone返回的token info中的user会不一样。当``impersonation=True``，token中的user信息是trustor。否则是当前发起request的用户。

# 总结

Trust最开始的目标客户是Heat，使得在Heat中的service user（heat）可以随时使用user的role操作user的资源。就我所知，目前Zaqar也使用了trust，使得一个队列的订阅者也可以是另外一个队列（更多细节可以另开Blog讲解）。更多的use case还需要用户去开发，大家如果有对Trust的需求或意见，可以找我一起探讨，也欢迎提交到OpenStack社区。
