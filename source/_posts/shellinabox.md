---
title: 通过浏览器SSH登录Linux
date: 2020-03-10 16:54:20
tags:
  - Shell
  - Linux
categories: Linux
---
1. 安装

```
$ apt install shellinabox
```

2. 修改配置

```
$ vi /etc/default/shellinabox
  SHELLINABOX_PORT=xxx # 自定义端口
  SHELLINABOX_ARGS="--no-beep --disable-ssl-menu --disable-ssl"
```

3. 重启服务

```
$ service shellinabox restart
```

4. 在VM的安全组中打开对应的端口（华为云）
5. 通过http://ip:port访问页面，登录VM

**Note**： 

1. shellinabox默认不支持root登录
2. 使用命令shellinaboxd -h获取更多信息
