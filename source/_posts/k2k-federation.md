---
title: Keystone实战-Keystone to Keystone Federation 1 
date: 2018-04-26 12:24:12
tags:
  - OpenStack
  - Keystone
categories: OpenStack
---
# 准备工作

1. 两台Ubuntu16.04机器，分别使用devstack部署Keystone服务。Keystone默认以uwsgi的方式托管在Apache下面。本文的Keystone是2018.04.25的Master代码，保证最新。

2. 根据这两台机器的IP，添加域名映射到``/etc/hosts``。

   ```
   10.3.150.135    idphost
   10.3.150.136    sphost
   ```
<!-- more -->
# 配置

## Keystone as SP

1. 安装libapache2-mod-shib2

   ```
   sudo apt install -y libapache2-mod-shib2
   ```

   生成公私钥对

   ```
   sudo shib-keygen -f
   ```

   启用服务

   ```
   sudo a2enmod shib2
   ```

2. 配置shibboleth

   修改文件``/etc/shibboleth/attribute-map.xml``：

   ```
   新增几个属性：
       <Attribute id="openstack_project" name="openstack_project"/>
       <Attribute id="openstack_project_domain" name="openstack_project_domain"/>
       <Attribute id="openstack_roles" name="openstack_roles"/>
       <Attribute id="openstack_user" name="openstack_user"/>
       <Attribute id="openstack_user_domain" name="openstack_user_domain"/>
   ```

   修改``/etc/shibboleth/shibboleth2.xml``：

   ```
   修改几个地方：
   ...
   # SP的名字
   <ApplicationDefaults entityID="KeystoneServiceProvider">
   ...
   # IdP的remote_id
   <SSO entityID="http://idphost/identity/v3/OS-FEDERATION/mapped/idp" ECP="true">
   ...
   # IdP的saml metadata URL
   <MetadataProvider type="XML" uri="http://idphost/identity/v3/OS-FEDERATION/saml2/metadata"
                     backingFilePath="metadata.xml" reloadInterval="180000" />
   ...
   ```

   重启shibboleth：

   ```
   sudo service shibd restart
   ```

3. 启用Apache重导向。

   修改Apache文件``/etc/apache2/sites-available/keystone-wsgi-public.conf``，文件最后加入如下信息。

   ```
   ProxyPass /Shibboleth.sso !

   <Location /Shibboleth.sso>
       SetHandler shib
   </Location>

   <Location /identity/v3/OS-FEDERATION/identity_providers/KeystoneIdP/protocols/mapped/auth>
       ShibRequestSetting requireSession 1
       AuthType shibboleth
       ShibExportAssertion Off
       Require valid-user

       <IfVersion < 2.4>
           ShibRequireSession On
           ShibRequireAll On
      </IfVersion>
   </Location>
   ```

   重启Apache服务

   ```
   service apache2 restart
   ```

4. 确保如下配置项在keystone.conf中配置正确：

   ```
   ...
   [auth]
   # methods中要包含mapped
   methods = ...., mapped
   ...
   [mapped]
   # federation的识别符，表示使用mod-shib2这种方式。
   remote_id_attribute = Shib-Identity-Provider
   ...
   ```

5. 创建Keystone的对应资源。

   创建新的domain、group并绑定角色：

   ```
   openstack domain create federated_domain
   openstack project create federated_project --domain federated_domain
   openstack group create federated_users --domain federated_domain
   openstack role add --group federated_users --domain federated_domain Member
   openstack role add --group federated_users --project federated_project Member
   ```

   创建Mapping：

   ```
   cat > rules.json <<EOF
   [
       {
           "local": [
               {
                   "user": {
                       "name": "{0}"
                   },
                   "group": {
                       "domain": {
                           "name": "federated_domain"
                       },
                       "name": "federated_users"
                   }
               }
           ],
           "remote": [
               {
                   "type": "openstack_user"
               }
           ]
       }
   ]
   EOF

   openstack mapping create --rules rules.json KeystoneIdP_mapping
   ```

   创建Identity Provider：

   ```
   openstack identity provider create KeystoneIdP --remote-id http://idphost/identity/v3/OS-FEDERATION/mapped/idp
   ```

   创建Protocol：

   ```
   openstack federation protocol create mapped --mapping KeystoneIdP_mapping --identity-provider KeystoneIdP
   ```

## Keystone as IdP

1. 安装依赖：

   ```
   sudo apt install -y xmlsec1
   sudo pip install pysaml2
   ```

2. 创建秘钥和证书：

   ```
   openssl req -x509 -newkey rsa:2048 -keyout /etc/keystone/keystone-saml-key.pem -out /etc/keystone/keystone-saml-cert.pem -days 9999 -nodes
   ```

3. 配置Keystone：

   在keystone.conf中：

   ```
   [saml]
   certfile=/etc/keystone/keystone-saml-cert.pem
   keyfile=/etc/keystone/keystone-saml-key.pem
   idp_entity_id=http://idphost/identity/v3/OS-FEDERATION/mapped/idp
   idp_sso_endpoint=http://idphost/identity/v3/OS-FEDERATION/saml2/sso
   idp_metadata_path=/etc/keystone/keystone_idp_metadata.xml
   ```

   重启Keystone：

   ```
   service devstack@keystone restart
   ```

4. 生成metadta文件。

   ```
   keystone-manage saml_idp_metadata > /etc/keystone/keystone_idp_metadata.xml
   ```

5. 创建Service Provider：

   ```
   openstack service provider create --auth-url http://sphost/identity/v3/OS-FEDERATION/identity_providers/KeystoneIdP/protocols/mapped/auth --service-provider-url http://sphost/Shibboleth.sso/SAML2/ECP KeystoneServiceProvider
   ```

# 测试

测试脚本：

```python
import json
import os

import requests

from keystoneclient import session as ksc_session
from keystoneclient.auth.identity import v3
from keystoneclient.v3 import client as keystone_v3


class K2KClient(object):
    def __init__(self):
        self.sp_id = 'KeystoneServiceProvider'
        self.auth_url = 'http://idphost/identity/v3'
        self.project_id = 'eb5136b98e7947e5b9537ea094871efa'
        self.username = 'admin'
        self.password = 'root'
        self.domain_id = 'default'

    def v3_authenticate(self):
        auth = v3.Password(auth_url=self.auth_url,
                           username=self.username,
                           password=self.password,
                           user_domain_id=self.domain_id,
                           project_id=self.project_id)
        self.session = ksc_session.Session(auth=auth, verify=False)
        self.session.auth.get_auth_ref(self.session)
        self.token = self.session.auth.get_token(self.session)
    def _generate_token_json(self):
        return {
            "auth": {
                "identity": {
                    "methods": [
                        "token"
                    ],
                    "token": {
                        "id": self.token
                    }
                },
                "scope": {
                    "service_provider": {
                        "id": self.sp_id
                    }
                }
            }
        }

    def _check_response(self, response):
        if not response.ok:
            raise Exception("Something went wrong, %s" % response.__dict__)

    def get_saml2_ecp_assertion(self):
        token = json.dumps(self._generate_token_json())
        url = self.auth_url + '/auth/OS-FEDERATION/saml2/ecp'
        r = self.session.post(url=url, data=token, verify=False)
        self._check_response(r)
        self.assertion = str(r.text)

    def _get_sp(self):
        url = self.auth_url + '/OS-FEDERATION/service_providers/' + self.sp_id
        r = self.session.get(url=url, verify=False)
        self._check_response(r)
        sp = json.loads(r.text)[u'service_provider']
        return sp

    def _handle_http_302_ecp_redirect(self, response, location, **kwargs):
        return self.session.get(location, authenticated=False, **kwargs)

    def exchange_assertion(self):
        """Send assertion to a Keystone SP and get token."""
        sp = self._get_sp()

        r = self.session.post(
            sp[u'sp_url'],
            headers={'Content-Type': 'application/vnd.paos+xml'},
            data=self.assertion,
            authenticated=False,
            redirect=False)

        self._check_response(r)

        r = self._handle_http_302_ecp_redirect(r, sp[u'auth_url'],
                                               headers={'Content-Type':
                                               'application/vnd.paos+xml'})
        self.fed_token_id = r.headers['X-Subject-Token']
        self.fed_token = r.text

    def list_federated_projects(self):
        url = 'http://sp:5000/v3/OS-FEDERATION/projects'
        headers = {'X-Auth-Token': self.fed_token_id}
        r = requests.get(url=url, headers=headers)
        self._check_response(r)
        return json.loads(str(r.text))

    def _get_scoped_token_json(self, project_id):
        return {
            "auth": {
                "identity": {
                    "methods": [
                        "token"
                    ],
                    "token": {
                        "id": self.fed_token_id
                    }
                },
                "scope": {
                    "project": {
                        "id": project_id
                    }
                }
            }
        }

    def scope_token(self, project_id):
        # project_id can be select from the list in the previous step
        token = json.dumps(self._get_scoped_token_json(project_id))
        url = 'http://sp:5000/v3/auth/tokens'
        headers = {'X-Auth-Token': self.fed_token_id,
                   'Content-Type': 'application/json'}
        r = requests.post(url=url, headers=headers, data=token,
                          verify=False)
        self._check_response(r)
        self.scoped_token_id = r.headers['X-Subject-Token']
        self.scoped_token = str(r.text)


def main():
    client = K2KClient()
    import pdb
    pdb.set_trace()
    client.v3_authenticate()
    client.get_saml2_ecp_assertion()
    print('ECP wrapped SAML assertion: %s' % client.assertion)
    client.exchange_assertion()
    print('Unscoped token id: %s' % client.fed_token_id)
    
    # If you want to get a scope token, please ensure federated_user has a project
    # and uncommen below codes.
    '''
    projects = client.list_federated_projects()
    print('Federated projects: %s' % projects['projects'])
    project_id = projects['projects'][0]['id']
    project_name = projects['projects'][0]['name']
    client.scope_token(project_id)
    print('Scoped token of ' + project_name + ' : ' + client.scoped_token_id)
    '''


if __name__ == "__main__":
    main()

```

测试流程：

1. 获取IdP中admin用户的scoped token
2. 用该token在IdP中获取SP需要的saml assertion。
3. 访问SP，校验该assertion
4. 使用该assertion在SP中生成unscoped token。

# 总结

Keystone 的Federation配置复杂，任何一步操作不当就可能会引起一系列错误，读者在操作的时候需要仔细。容易出错的地方在于IdP、SP、shibboleth、Apache四者之间数据的配置不一致。

Federation一般常被WebUI封装，通过websso方式实现页面中IDP与SP之间的自动跳转。我将在下一篇文章中介绍如何配置Keysont二和Horizon，使两者串起来，实现websso。

