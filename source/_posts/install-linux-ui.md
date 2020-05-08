---
title: 在Ubuntu Server 18.04上安装UI+VNC登录  
date: 2020-05-08 15:20:03
tags:
  - Linux
categories: Linux 

---

# 步骤
1. 安装VNC和UI软件

    ```
    apt install vnc4server
    apt install xfce4
    ```
<!-- more -->
2. 修改文件`~/.vnc/xstartup`

    ```
    #!/bin/sh
    unset SESSION_MANAGER
    unset DBUS_SESSION_BUS_ADDRESS
    
    [ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
    [ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
    xsetroot -solid grey
    
    vncconfig -iconic &
    
    xfce4-session & startxfce4 & 
    
    x-terminal-emulator -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &
    
    sesion-manager & xfdesktop & xfce4-panel &
    xfce4-menu-plugin &
    xfsettingsd &
    xfconfd &
    xfwm4 &
    ```

    P.S. 如果没有该文件，则先启动一次vnc4server，然后stop掉。

    ```
    $ vnc4server
    输入VNC初始密码
    $ vnc4server -kill:1
    ```

    如果还没有，那就自己手动创建一个新文件，记得加上`+x`权限。

3. 安装必须的字体

    ```
    apt install xfonts-100dpi xfonts-75dpi xfonts-scalable xfonts-cyrillic
    ```

    创建目录`/usr/X11R6/lib/X11/fonts`，并把`/usr/share/fonts/X11`目录下的全部`文件/文件夹`拷贝到`/usr/X11R6/lib/X11/fonts`

4. 再次执行vnc4server

    ```
    $ vncserver -depth 16 -geometry 1920x1080  (根据自己的屏幕大小修改分辨率)
    ```

5. 查看vnc对应的port，防火墙打开该port。

    ```
    $ netstat -nltp|grep vnc
    ```

    vnc4server一般安装5901、5902、5903...的顺序起port，因此第一个vnc服务的port一般是5901。
    
6. 使用vnc client登录
    VNC client五花八门，选择一个适合自己的就行。本人这里由于公司限制，只能使用专门的VNC软件，就不详细介绍了。登录信息一般包括：IP， port，Password（记得第一启动vnc4server时输入的密码吗，就是那个）。
