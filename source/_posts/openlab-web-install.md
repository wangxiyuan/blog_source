---
title: OpenLab 主页搭建指南 
date: 2019-10-15 02:05:08
tags:
  - OpenLab
  - Web
categories: OpenLab
---

# 准备

1. Ubuntu 可通外网的VM一台。
2. 已购域名一个。
3. 安全组开放80、443、3306端口。
<!-- more -->

# 安装

1. apche2以及相关mod，例如mod_php、mod_rewrite、mod_ssl
2. mysql
3. php7.0以及相关组件，例如php7.0-mysql、php7.0-curl等。
4. 下载wordpress，解压到/var/www目录。

# 配置

1. 去域名提供商处添加DNS解析，使域名指向VM的公网IP。

2. mysql中创建wordpress数据库。

3. `/etc/apache2/apache2.conf`中添加

   ```
   <Directory /var/www/wordpress>
           Options Indexes FollowSymLinks
           AllowOverride All
           Order allow,deny
           Allow from all
   </Directory>
   ```

4. `/etc/apache2/sites-available/000-default.conf`中修改：

   ```
   DocumentRoot /var/www/wordpress
   ServerName OpenLab
   RewriteEngine on
   ```

5. 新增`/etc/apache2/sites-available/openlab-ssl.conf`，内容与`000-default.conf`基本一致，不同点：

   ```
    <VirtualHost *:443>
    SSLEngine on
    SSLCertificateFile      /etc/apache2/www_openlabtesting_org.pem
    SSLCertificateKeyFile   /etc/apache2/www_openlabtesting_org.key
   ```

   其中证书可以去let's encrypt免费申请。OpenLab使用的域名提供商Dnsimple自动集成了Let's encrypt，直接去Dnsimple下载即可。

6. 创建`/var/www/wordpress/wp-content/tmp`目录，修改`/var/www/wordpress/wp-config.php`，新增

   ```
   define('WP_TEMP_DIR',ABSPATH.'wp-content/tmp');
   define("FS_METHOD","direct");
   define("FS_CHMOD_DIR",0777);
   define("FS_CHMOD_FILE",0777);
   ```

7. 访问网站域名，根据wordpres提示，安装网站。
8. 登陆`https://{domain_name}/wp-admin`，后台设计、配置wordpress网站。
