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
Ubuntu：
$ apt install shellinabox
其他系统请自行安装
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

4. 在VM的安全组中打开对应的端口，如果VM本身配置了防火墙，也需要放开对应端口。（我使用的是华为云）
5. 通过http://ip:port 访问页面，登录VM。

**Note**： 

1. shellinabox默认不支持root登录。
2. 使用命令`shellinaboxd -h`获取更多信息。
3. 如果想使用https方式访问，请删掉`disable ssl`相关的配置，并配置对应的CA等文件,`shellinaboxd -h`中有详细说明。
