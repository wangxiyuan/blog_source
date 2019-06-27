---
title: Keystone中的联邦认证机制（Federation Identity）
date: 2017-10-30 14:38:20
tags:
  - OpenStack
  - Keystone
categories: OpenStack
---
所谓OpenStack的联邦鉴权就是指通过Keystnoe与第三方服务配合，实现使用第三方鉴权信息登录现有云环境的机制。例如，用户A想使用公有云X，但出于安全考虑，不想把自己的信息放在X上，通过federation identity打通二者之间的认证，A只需要在可信任的地方Y（可以是其他云、第三方认证系统等等）上完成鉴权校验，就可以访问X。还有我们在使用软件时，常常用的“使用微信登录”这样的功能，也是基于federation identity的。Keystone federation identity为混合云在用户管理层面提供了良好的解决方案。在讲述Keystone的联邦鉴权之前，我们需要先理解几个关键的概念：
<!-- more -->
1. Identity Provider（IdP）：鉴权提供方，又称断言（assertion）方，即存储并校验用户鉴权信息的地方。如上例的Y。常见的有ADFS(Active Directory Federation Services)、FreeIPA、Tivoli Access Manager等。
2. Service Provider（SP）：服务提供方，只提供服务，不包含鉴权部分。如上例的X。
3. Assertion Protocol:认证协议，即IdP和SP之前协作时用到的方法，常用的有SAML、OpenID、Oauth等。由IdP生成，如IdP使用SAML方式向SP完成校验。
4. Web Single Sign On（Web SSO）：网络单点登录，用户可以通过一个账号，登录不同的系统或服务，如上例中通过登录Y，来登录X。

以及Keystone中用到的概念： 
1. Mapping：一组SP合IdP之间的映射规则，如IdP中的A用户在SP中有什么样的权限等。IdP中每个Protocol对应一个Mapping对象。
2. Unscoped token: 没有作用域的token，这种token只有用户信息，不包含角色、租户、域、endpoint信息，因此只能被Keystone使用，用来获取对应的scope（作用域），从而生成对应的scoped token
3. Scoped token：包含了所有必须的信息，被其他服务接收并使用。

# 1. Keystone Federation的原理

Keystone是IdP，还是SP？
- 场景1：华为公有云（HWS）使用的是OpenStack架构，向外提供了Keystone服务。公司A在本地使用了LDAP类的系统存储本公司员工的用户信息。现在公司A的员工B想通过Federation identity机制，用公司的LDAP中自己的信息登录HWS。此时公司A的LDAP是LdP，HWS的Keystone是SP。
- 场景2：公司A在本地使用OpenStack部署了一套私有云，现在想用本地Keystone的用户通过Federation identity机制登录AWS（假设AWS支持了该方法）。此时本地的Keystone是IdP，AWS的IAM服务是SP。
- 场景3：公司A在本地使用OpenStack部署了一套私有云，现在想用本地Keystone的用户通过Federation identity机制登录HWS。此时本地的Keystone是IdP，HWS的Keystone是SP。

因此，针对不同场景，Keystone既可以是IdP，也可以是SP。下面看一张经典的Federation流程图:
![Federation Example](/images/keystone-federation1.png)

1. 用户发送请求给SP，请求查看服务列表。
2. SP验证用户的有效性，根据用户请求中包含的验证方法，SP服务将信息发送至第三方IdP，进一步请求用户身份验证。
3. 第三方IdP验证用户身份后，发送至SP，证明用户身份已经确认，里面包含用户的具体信息及IdP的具体信息。
4. SP对claim信息进行检查，确认IdP身份有效后，查询本地数据中IdP与SP之间对于用户的映射表，找出映射到的用户组。
5. SP根据对应的用户组，验证用户的操作权限，并返回此权限范围内可看到的云环境中的服务列表。

## 1.1 Keystone as IdP

Keystone作为Identity时，只需要接收SP的鉴权请求，生成对应的Protocal返回给SP即可。

## 1.2 Keystone as SP

Keystone作为Service Provider，当用户访问Keystone想请求一个Token时，Keystone首先把鉴权请求转向对应的Identity Provider，然后接收并校验IdP传来的Protocol内容，最后根据Mapping关系把该内容转化成一个Token返回给用户。

# 2. Keystone Federation的配置

下面我们看看如何配置Keystone使之成为IdP或SP

## 2.1 Keystone as IdP

作为IdP，关键点在于如何生成对应的Protocal，以SAML为例，Keystone使用PySAML2库来生成SAML。生成SAML时，需要在配置文件中新增相关的配置项。关键的几个如下：
>[saml]
> 生成SAML时用到的公私钥，需要管理员提供。
>certfile=/etc/keystone/ssl/certs/ca.pem
>keyfile=/etc/keystone/ssl/private/cakey.pem
>
> SAML的生产者的一些必要信息。
>idp_entity_id=https://keystone.example.com/v3/OS-FEDERATION/saml2/idp
>idp_sso_endpoint=https://keystone.example.com/v3/OS-FEDERATION/saml2/sso
>
> 生产SAML需要用到的metadata文件，用``keystone-manage saml_idp_metadata``命令生成。
>idp_metadata_path=/etc/keystone/saml2_idp_metadata.xml

还有一些可选配置项,是一些SAML生产者的补充信息：
>idp_organization_name=example_company
>idp_organization_display_name=Example Corp.
>idp_organization_url=example.com
>idp_contact_company=example_company
>idp_contact_name=John
>idp_contact_surname=Smith
>idp_contact_email=jsmith@example.com
>idp_contact_telephone=555-55-5555
>idp_contact_type=technical

配置完后，还需要在Keystone中建立与SP的联系，即在Keystone中创建SP对象。CLI操作如下：
>openstack service provider create test_sp_name --service-provider-url XXX --auth-url YYY

其中test_sp_name是SP的名字。service-provider-url是sp的url，在生成SAML时使用。auth-url是指生成后的SAML发向何处。
至此Keystone as IdP配置完成。当test_sp_name发来一个鉴权请求时，Keystone通过以上配置生成SAML，然后把SAML发向test_sp_name的auth-url。

## 2.2 Keystone as SP

Keystone作为Service Provider时，需要做以下配置：

1. 新增mapped鉴权方式，表示允许接收federation类的认证，如允许接收SAML。
>[auth]
>mapped是saml2（SAML的2.0版本）和openid的统称
>methods = external,password,token,mapped
2. 在Keystone中，创建专用做federation的用户或用户组（local_user or local_group）。
3. 在Keystone中创建IdP对象。remote-ids表示该IdP对应的id，用来防止其他IdP侵入。
>openstack identity provider create test_idp_name --remote-ids xxx,yyy...
4. 在Keystone中创建Mapping。rules是指IdP中的user（叫作remote_user)与local_user或local_group之间的映射关系，涉及到权限等。在附录中会对其语法进行详细说明。
cal_user or local_group
>openstack mapping create test_mapping_name --rules xxxx
5. 在Keystone中创建对应的protocol。其中identity-provider是第三步创建的IdP对象；mapping是第四步创建的规则。
>openstack federation protocol create test_protocol_name --identity-provider xxx --mapping yyy
6. 把Keystone部署在Apache下，并启用libapache2-mod-shib2模块并进行相关配置。该模块会对请求进行重转向。
>
至此，Keystone as SP配置完成。用户访问Keystone时，Apache把请求转向IdP。去IdP认证完后，请求带着生成的SAML继续访问Keystone，Keystone就会返回一个unscoped token给用户。然后用户用这个unscoped token访问Keystone，获取一个scoped token。最后用户就可以用这个token访问其他服务啦。

# 3. Keystone to Keystone Federation

了解了What和How后，我们下来通过一个例子重点来讲解Why。
>实验环境：两套OpenStack，Openstack-A中的Keystone-A作为Service Provider，Openstack-B中的Keystone-B作为Identity Provider。Keystone-A、B中分别有user-A、B。
>实验目标：Keystone-B中的user-B访问Keystone-A后，拥有user-A的权限，可以访问Keystone-A中的Nova、Neutron、Cinder等服务。

Keystone-A的准备：
1. 当用户B访问Keystone-A时，Keystone-A需要有重导向Keystone-B的能力，这里必须把Keystone-A部署在Apache下，使用Apache的扩展模块实现该功能。目前支持的有：
  - mod-shib2(SAML协议)，本文以此为例
  - mod_auth_mellon（SAML协议）
  - mod_auth_openidc （OpenID协议）

  步骤如下：
    > 1. apt-get install libapache2-mod-shib2
    > 2. 编辑/etc/apache2/sites-available/keystone.conf，在<VirtualHost *:5000>（因环境而异，根据Keystone部署方式不同而不同，例如O版的devstack就可以如此，但P版之后devstack的Keystone默认使用uwsgi启动，就不能直接这么做了。）下新增一行WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /usr/local/bin/keystone-wsgi-public/$1。并新增对应的IdP的Location。
```
<VirtualHost *:5000>  
    WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /usr/local/bin/keystone-wsgi-public/$1

<Location /Shibboleth.sso>
    SetHandler shib
</Location>

<Location /v3/OS-FEDERATION/identity_providers/myidp/protocols/mapped/auth>
    ShibRequestSetting requireSession 1
    AuthType shibboleth
    ShibExportAssertion Off
    Require valid-user

    <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
   </IfVersion>
</Location>
...
```
    >   ``mapped``是后续要创建的protocol类型。
    >   ``myidp``是后续要创建的IdP的名字。
    > 3. 打开shib2模块，并重启Apache。
```
$ a2enmod shib2
$ service apache2 restart
```
    > 4. 生成keypair，编辑XML。这一步是为了生成shibboleth的metadata。细节不再表述，请参考： https://docs.openstack.org/keystone/pike/advanced-topics/federation/shibboleth.html

2. 配置Keystone-A。
  1. 在配置项methods中新增mapped。
     > [auth]
     > methods = external,password,token,mapped
  2. 创建Federation专用的domain、project、group和user-A并赋予相应的权限。
```
$ openstack domain create federated_domain
$ openstack project create federated_project --domain federated_domain
$ openstack user create user-A --domain federated_domain
$ openstack group create federated_users
$ openstack role add --group federated_users --domain federated_domain Member
$ openstack role add --group federated_users --project federated_project Member
$ openstack role add --user user-A --domain federated_domain Member
$ openstack role add --user user-A --project federated_project Member
```
  3. 创建IdP对象。
```
$ openstack identity provider create --remote-id https://Keystone-B/v3/OS-FEDERATION/saml2/idp myidp

```
     ``remote-id``前面讲过，是IdP的唯一标识，由IdP的metadata提供。这个例子中IdP是Keystone-B，它的``remote-id``由配置项[saml]/idp_entity_id提供。
  4. 创建Mapping。映射User-A和User-B的关系
```
$ cat > rules.json <<EOF
[
    {
        "local": [
            {
                "user": {
                    "name": "User-A"
                },
                "group": {
                    "domain": {
                        "name": "Default"
                    },
                    "name": "federated_users"
                }
            }
        ],
        "remote": [
            {
                "type": "openstack_user",
                "any_one_of": [
                    "User-B"
                ]
            }
        ]
    }
]
EOF
$ openstack mapping create --rules rules.json myidp_mapping
```
  5. 创建对应的protocol
``` 
$ openstack federation protocol create mapped --mapping myidp_mapping --identity-provider myidp
```

Keysonte-B的准备：
1. 配置[smal]的众多配置项，注意与SP配置的对应关系。并生成metadata。
2. 创建user-B。
3. 创建SP对象。
```
$ openstack service provider create --service-provider-url 'http:///keystone-A/Shibboleth.sso/SAML2/ECP' --auth-url http://Keystone-A:5000/v3/OS-FEDERATION/identity_providers/myidp/protocols/mapped/auth mysp
```

整体流程：
1. User-B访问keystone-A请求Token，POST /v3/OS-FEDERATION/identity_providers/myidp/protocols/mapped/auth
2. Keysonte-A接收到请求，由Apache重导向到KeystoneB。
3. Keysotne-B对User-B鉴权通过，生成SAML，返回给Keysotne-A
4. Keystone-A校验SAML通过，由Mapping知道User-B有本地User-A的权限，生成User-A对应的unscoped token，返回给User-B
5. User-B带着这个unscoped token请求Keystone-A，获取对应的project作用域， GET /v3/OS-FEDERATION/project
6. User-B带着unscoped token和作用域，请求Keystone-A，获取scoped token。POST /V3/auth/token
7. User-B带着scoped token，访问Keystone-A所在的OpenStack。

# 4. 总结

Federation identity是Web架构下，应用非常广泛的一项技术。而在云场景下，Keysonte的此功能很好的解决了OpenStack多云互联的鉴权问题。Keystone还支持与Horizon配合，通过Ferderation支持Web Single Sign-On (单点登录，SSO)。并且Keystone还支持OpenID、Oauth等协议。但Keystone federation的配置与部署十分复杂，再加上开源社区不注重文档导致文档较少或过期的问题，使得学习门槛变高，本文也有很多细节没有涉及。更多细节还是推荐大家通过Keystone代码来理解。

# 附录：Mapping rule语法。

Mapping是一个Json体，以“rules”开头，形式如下：

```
{
    "rules": [mapping1,mapping2....mappingN]
}
```

其中每个mappingN也是一个Json体，各包含两个key：“local”和“remote”，形式如下： 

```
{
    "local": [xxx,yyy,zzz],
    "romote": [aaa,bbb,ccc]
}
```

local中包含SP的用户信息，可包括：user、project、domian以及group，形式如下：

```
[
    {
        "user": {
            "name": "{0}"
        },
        "group": {
            "id": "0cd5e9"
        }
    },
    {
        "user": {
            "id": "0cd5e8",
	    "domain": {
	        "id": "0cd5e7"
	    }
        }
    },
    {
        "group": {                        
            "name": "{1}",
	    "domain": {
	        "name": "test"
	    }
        }
    }
]
```

remote包含IdP的信息，以SAML协议为例，就是SAML中的一些信息。形式如下：

```
[
    {
        "type": "UserName"
    },
    {
        "type": "cn=IBM_Canada_Lab",
        "not_any_of": [
            ".*@naww.com$"
        ],
        "regex": true
    },
    {
        "type": "cn=IBM_USA_Lab",
        "any_one_of": [
            ".*@yeah.com$"
        ],
        "regex": true
    }
]
```

其中“not_any_of”和“any_one_of”是关键字，意思如字面所示。其他关键字还有：“blacklist”和“whitelist”。因此组合起来，这个rule就产生了:
```
{
    "rules": [
        {
            "local": [
                {
                    "user": {
                        "name": "{0}"
                    },
                    "group": {
                        "id": "0cd5e9"
                    }
                },
                {
                    "user": {
                        "id": "0cd5e8",
                        "domain": {
                            "id": "0cd5e7"
                        }
                    }
                },
                {
                    "group": {
                        "name": "{1}",
                        "domain": {
                            "name": "test"
                        }
                    }
                }
            ],
            "romote": [
                {
                    "type": "UserName"
                },
                {
                    "type": "cn=IBM_Canada_Lab",
                    "not_any_of": [
                        ".*@naww.com$"
                    ],
                    "regex": true
                },
                {
                    "type": "cn=IBM_USA_Lab",
                    "any_one_of": [
                        ".*@yeah.com$"
                    ],
                    "regex": true
                }
            ]

        }
    ]
}
```

可以使用Keystone自带的命令``keystone-manage mapping_engine``来验证rule：

```
$ keystone-manage mapping_engine --rules <file> --input <file>
```

其中rules是刚生成的JSON对象，input是IdP的SAML对象。如果Mapping与SAML匹配，则输出转换后的结果。
