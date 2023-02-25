# 自建内网穿透服务器

#### 介绍
几种通过自己服务器实现内网穿透的教程

## 基于Docker的FRP内网穿透部署

### 服务器搭建（FRPS）
创建配置文件
```
# 创建存放目录
sudo mkdir /etc/frp
# 创建frps.ini文件
nano /etc/frp/frps.ini
```
frps.ini内容如下：
```
[common]
# 监听端口
bind_port = 7000
# 面板端口
dashboard_port = 7500
# 登录面板账号设置
dashboard_user = admin
dashboard_pwd = spoto1234
# 设置http及https协议下代理端口（非重要）
vhost_http_port = 7080
vhost_https_port = 7081


# 身份验证
token = 12345678
```

```
#服务器镜像：snowdreamtech/frps
#重启：always
#网络模式：host
#文件映射：/etc/frp/frps.ini:/etc/frp/frps.ini

docker run --restart=always --network host -d -v /etc/frp/frps.ini:/etc/frp/frps.ini --name frps snowdreamtech/frps
```



### 中转客户端配置（FRPC）
```
服务器镜像：snowdreamtech/frpc
重启：always
网络模式：host
文件映射：/路径/frp/:/etc/frp/
```

配置文件示例：
```
[common]
# server_addr为FRPS服务器IP地址
server_addr = x.x.x.x
# server_port为服务端监听端口，bind_port
server_port = 7000
# 身份验证
token = 12345678

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 2288

# [ssh] 为服务名称，下方此处设置为，访问frp服务段的2288端口时，等同于通过中转服务器访问127.0.0.1的22端口。
# type 为连接的类型，此处为tcp
# local_ip 为中转客户端实际访问的IP 
# local_port 为目标端口
# remote_port 为远程端口

[ssh]
type = tcp
local_ip = 192.168.1.229
local_port = 80
remote_port = 18022

[unRAID web]
type = tcp
local_ip = 192.168.1.229
local_port = 80
remote_port = 18088

[Truenas web]
type = tcp
local_ip = 192.168.1.235
local_port = 80
remote_port = 18188

[speedtest]
type = tcp
local_ip = 192.168.1.229
local_port = 6580
remote_port = 18190


[webdav]
type = tcp
local_ip = 192.168.1.235
local_port = 18080
remote_port = 18189

[RDP PC1]
type = tcp
local_ip = 192.168.1.235
local_port = 3389
remote_port = 18389
```
- 如果监听服务可以有IP限制的设置，需要允许的访问IP为中转内网设备的内网IP；
- FRP由于端口会暴露在互联网上，虽然说使用方便但安全性较差；

## 基于Zerotier根服务器的内网穿透部署


### 创建（伪）根服务器 | 项目地址：https://github.com/Jonnyan404/zerotier-planet

```
docker run --restart=on-failure:3 -d --name ztncui -e HTTP_PORT=4000 -e HTTP_ALL_INTERFACES=yes -e ZTNCUI_PASSWD=mrdoc.fun -p 4000:4000 keynetworks/ztncui
```

### 创建 moon 服务器 | 项目地址：https://github.com/jonnyan404/docker-zerotier-moon
```
#创建容器
docker run --name zerotier-moon -d -p 9993:9993 -p 9993:9993/udp -v /etc/ztconf/:/var/lib/zerotier-one jonnyan404/zerotier-moon -4 [公网ipx.x.x.x]

#查看moon ID
docker logs zerotier-moon
```

### 群晖 DSM 7.x 安装Zerotier客户端

#### 登录SSH并创建虚拟网络设备TUN
```
#获取权限
sudo -i

#创建“创建虚拟网络设备TUN”的脚本，并设为开机自动运行
echo -e '#!/bin/sh -e \ninsmod /lib/modules/tun.ko' > /usr/local/etc/rc.d/tun.sh

#给予脚本运行权限
chmod a+x /usr/local/etc/rc.d/tun.sh

#运行脚本创建TUN
/usr/local/etc/rc.d/tun.sh

#确认TUN是否创建成功
ls /dev/net/tun
```

创建存放配置文件的目录
```
mkdir /var/lib/zerotier-one
```
创建Zerotier应用容器：
```
docker run -d           \
  --name zt             \
  --restart=always      \
  --device=/dev/net/tun \
  --net=host            \
  --cap-add=NET_ADMIN   \
  --cap-add=SYS_ADMIN   \
  -v /var/lib/zerotier-one:/var/lib/zerotier-one zerotier/zerotier-synology:latest
```

常用命令：
```
#查看zerotier状态
docker exec -it zt zerotier-cli status

#加入网络
docker exec -it zt zerotier-cli join [xxxxxxxxxxxx]
```

```
#加入moon服务器
docker exec zt zerotier-cli orbit [moon_ID] [moon_ID]
#确认是否加入
docker exec zt zerotier-cli listpeers 
```

### Windows 客户端加入moon服务器
```
cd C:\ProgramData\ZeroTier\One
zerotier-cli orbit [moon_id] [moon_id]
```
