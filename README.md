# 自建内网穿透服务器

#### 介绍
几种通过自己服务器实现内网穿透的教程

## 基于Ubuntu安装docker
卸载旧版本docker
```
sudo apt-get remove docker docker-engine docker.io containerd runc
```
更新软件包索引并安装软件包以允许使用 基于 HTTPS 的存储库：aptapt
```
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
添加 Docker 的官方 GPG 密钥
```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
使用以下命令设置存储库
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
更新包索引：apt
```
sudo apt-get update
```
运行时收到 GPG 错误？(apt-get update)
默认掩码可能配置不正确，导致无法检测到 存储库公钥文件。尝试授予 Docker 的读取权限 更新包索引之前的公钥文件
```
sudo chmod a+r /etc/apt/keyrings/docker.gpg
sudo apt-get update安装 Docker Engine、containerd 和 Docker Compose
```
安装最新的 Docker Engine、containerd 和 Docker Compose
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
通过运行映像验证 Docker 引擎安装是否成功：hello-world
```
sudo docker run hello-world
```

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
dashboard_pwd = admin23
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

服务端开放端口：
- WebUI：7500
- Sever: 7000
- Other: 7080,7081

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
```
- 如果监听服务可以有IP限制的设置，需要允许的访问IP为中转内网设备的内网IP；
- FRP由于端口会暴露在互联网上，虽然说使用方便但安全性较差；

## 基于Docker的Nginx-Proxy-Manager反向代理部署


### vps客户端配置

创建挂在目录
```
# 创建data存放目录
sudo mkdir /etc/npm/data
# 创建证书存放目录
sudo mkdir /etc/npm/letsencrypt
```
配置docker
```
docker run --restart=always --network host -d -v /etc/npm/data:/data -v /etc/npm/letsencrypt:/etc/letsencrypt --name npm chishin/nginx-proxy-manager-zh
#服务器镜像：chishin/nginx-proxy-manager-zh（中文汉化版本镜像）,jc21/nginx-proxy-manager（英文原版镜像）
#容器名：npm
#重启：always
#网络模式：host
#文件映射：数据data&ssl证书
```

服务端开放端口：
- WebUI：81 (后面可以配置反向代理，可以关闭防火墙)
- Http: 80
- Https: 443
