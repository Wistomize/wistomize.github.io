---
title: "使用Frp进行内网穿透"
date: 2026-02-20
description: "只要有一个公网IP能访问的服务器，就可以随时随地访问你的内网服务器"
tags: ['Linux', 'Frp', 'Server', 'SSH', 'apt', 'Docker', 'pyenv']
series: ["自建Linux服务器实录"]
---

如果你的自建服务器在内网，没有一个公网可以访问的IP地址，不要慌，只要随便找一个能从公网访问的服务器，跟着流程走就可以了。如果确实没有，那就从云服务商那里薅一点便宜羊毛，买那种最便宜的服务器（哪怕是1U2G的规格都可以，只要有个公网IP就行），然后照着来就行。

本教程为全流程无坑标准化部署方案，适用于公网 Ubuntu 云服务器 + 内网 Ubuntu 服务器，可实现 SSH 远程登录、内网 WEB 服务、内网大模型 API 等场景的公网穿透访问。

## 一、前置准备与环境说明

### 1.1 必备环境

- 一台拥有**公网固定IP**的云服务器（操作系统：Ubuntu/Debian 系列，本文以 Ubuntu 为例）

- 一台待穿透的内网服务器（操作系统：Ubuntu，可正常访问公网）

- 本地电脑已配置 SSH 密钥对，可正常远程登录公网云服务器

### 1.2 端口规划

先想想你自己的服务器有哪些服务需要对外开放，以下为举例：

|端口号|协议|核心用途|
|---|---|---|
|7000|TCP|frp 服务端与客户端核心通信端口|
|6000|TCP|内网服务器 SSH 服务穿透端口|
|8080|TCP|内网 WEB 服务/管理面板穿透端口|
|8000|TCP|内网大模型 API/应用服务穿透端口|
---

## 二、公网云服务器（frp 服务端）部署

全程使用 SSH 登录公网服务器操作，优先使用普通 sudo 用户，避免直接使用 root 账号。

### 2.1 创建专属 sudo 管理用户

```Bash

# 1. 创建专属管理用户（示例用户名为 frpadmin，可自定义）
sudo adduser frpadmin

# 2. 将用户加入 sudo 组，授予管理员权限
sudo usermod -aG sudo frpadmin

# 3. 切换到该用户，后续操作均在此用户下执行
su - frpadmin
```

### 2.2 端口放行配置（云安全组 + 系统防火墙）

#### 第一步：云控制台安全组放行

登录你的云服务商控制台（阿里云/腾讯云/华为云等），进入服务器「安全组」设置，在**入方向规则**中添加以下TCP端口放行规则：

- 端口范围：7000、6000、8080、8000

- 协议类型：TCP

- 源地址：临时填写 `0.0.0.0/0`（后续安全加固可修改为个人常用IP）

- 策略：允许

#### 第二步：Ubuntu 系统防火墙配置（ufw）

```Bash

# 1. 优先放行 SSH 默认22端口，避免远程连接断开
sudo ufw allow 22/tcp

# 2. 放行 frp 所需核心端口
sudo ufw allow 7000/tcp
sudo ufw allow 6000/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 8000/tcp

# 3. 开启防火墙并重载规则生效
sudo ufw enable
sudo ufw reload

# 4. 验证规则是否生效
sudo ufw status
```

### 2.3 下载并安装 frp 服务端

本教程使用 frp v0.58.1 稳定版，服务端与客户端需保持版本完全一致。

```Bash

# 1. 更新系统依赖
sudo apt update -y

# 2. 下载 frp 安装包（amd64 架构，云服务器通用）
wget https://github.com/fatedier/frp/releases/download/v0.58.1/frp_0.58.1_linux_amd64.tar.gz

# 3. 解压安装包
tar -zxvf frp_0.58.1_linux_amd64.tar.gz

# 4. 进入解压目录
cd frp_0.58.1_linux_amd64

# 5. 复制 frps 服务端主程序到系统全局路径
sudo cp frps /usr/local/bin/

# 6. 创建 frp 专属配置文件目录
sudo mkdir -p /etc/frp

# 7. 复制默认配置文件到专属目录
sudo cp frps.toml /etc/frp/

# 8. 验证安装是否成功（输出版本号即正常）
frps --version
```

### 2.4 编辑 frp 服务端核心配置文件

```Bash

# 用 nano 编辑器打开配置文件
sudo nano /etc/frp/frps.toml
```

清空文件原有内容，复制粘贴以下标准化配置，**仅需修改 auth.token 为自定义强密码，其余配置无需改动**：

```TOML

# ====================== 基础连接配置 ======================
bindPort = 7000
bindAddr = "0.0.0.0"

# ====================== 核心身份认证 ======================
auth.method = "token"
# 自定义强密码，客户端配置必须与此处完全一致！
auth.token = "Frp@2026#YourStrongToken123"
auth.additionalScopes = ["HeartBeats", "NewWorkConns"]

# ====================== 传输安全加固 ======================
transport.tls.force = true
transport.maxPoolCount = 5

# ====================== 端口与权限限制 ======================
maxPortsPerClient = 5
allowPorts = [
  { start = 6000, end = 6000 },
  { start = 8000, end = 8000 },
  { start = 8080, end = 8080 }
]

# ====================== 日志配置 ======================
log.to = "/var/log/frps.log"
log.level = "info"
log.maxDays = 7
```

保存并退出编辑器：按 `Ctrl+O` → 按 `Enter` 确认 → 按 `Ctrl+X` 退出。

### 2.5 配置 frps 系统服务（开机自启+崩溃重启）

```Bash

# 1. 创建 systemd 服务文件
sudo nano /etc/systemd/system/frps.service
```

复制粘贴以下服务配置，无需修改任何内容：

```TOML

[Unit]
Description=Frp Server Service
After=network.target syslog.target
Wants=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5s
User=root

[Install]
WantedBy=multi-user.target
```

保存并退出编辑器后，执行以下命令启动服务：

```Bash

# 2. 重载 systemd 配置，识别新增服务
sudo systemctl daemon-reload

# 3. 启动 frps 服务
sudo systemctl start frps

# 4. 设置服务开机自启
sudo systemctl enable frps

# 5. 验证服务状态（显示 active (running) 即部署成功）
sudo systemctl status frps
```

---

## 三、内网 Ubuntu 服务器（frp 客户端）部署

全程使用内网服务器的 sudo 用户操作，确保内网服务器可正常访问公网。

### 3.1 前置网络连通性验证

```Bash

# 1. 验证与公网服务器的网络连通性（替换为你的公网服务器IP）
ping *.*.*.*

# 2. 验证公网服务器7000端口可正常访问（替换为你的公网服务器IP）
telnet *.*.*.* 7000
```

能正常 ping 通、telnet 连接成功，即可进入后续步骤。

### 3.2 下载并安装 frp 客户端

与服务端保持 v0.58.1 版本完全一致：

```Bash

# 1. 下载 frp 安装包
wget https://github.com/fatedier/frp/releases/download/v0.58.1/frp_0.58.1_linux_amd64.tar.gz

# 2. 解压安装包
tar -zxvf frp_0.58.1_linux_amd64.tar.gz

# 3. 进入解压目录
cd frp_0.58.1_linux_amd64

# 4. 复制 frpc 客户端主程序到系统全局路径
sudo cp frpc /usr/local/bin/

# 5. 创建 frp 专属配置文件目录
sudo mkdir -p /etc/frp

# 6. 复制默认配置文件到专属目录
sudo cp frpc.toml /etc/frp/

# 7. 验证安装是否成功（输出版本号即正常）
frpc --version
```

### 3.3 编辑 frp 客户端核心配置文件

```Bash

# 用 nano 编辑器打开配置文件
sudo nano /etc/frp/frpc.toml
```

清空文件原有内容，复制粘贴以下标准化配置，**必须修改3处核心参数：serverAddr、auth.token、对应服务的localPort**，其余配置无需改动：

```TOML

# ====================== 连接公网服务器配置 ======================
# 替换为你的公网服务器固定IP
serverAddr = "8.141.11.115"
serverPort = 7000

# ====================== 身份认证（必须与服务端完全一致！） ======================
auth.method = "token"
# 替换为服务端配置的完全相同的token密码
auth.token = "Frp@2026#YourStrongToken123"
auth.additionalScopes = ["HeartBeats", "NewWorkConns"]

# ====================== 传输安全配置（与服务端匹配） ======================
transport.tls.enable = true

# ====================== 日志配置 ======================
log.to = "/var/log/frpc.log"
log.level = "info"
log.maxDays = 7

# ====================== 穿透规则配置（3个核心场景） ======================
# 1. 内网SSH服务穿透
[[proxies]]
name = "inner-linux-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000

# 2. 内网WEB服务穿透
[[proxies]]
name = "inner-web-service"
type = "tcp"
localIP = "127.0.0.1"
# 替换为你内网WEB服务的实际端口
localPort = 80
remotePort = 8080

# 3. 内网大模型API/应用服务穿透
[[proxies]]
name = "inner-llm-api"
type = "tcp"
localIP = "127.0.0.1"
# 替换为你内网API服务的实际端口
localPort = 8000
remotePort = 8000
```

保存并退出编辑器：按 `Ctrl+O` → 按 `Enter` 确认 → 按 `Ctrl+X` 退出。

### 3.4 配置 frpc 系统服务（开机自启+崩溃重启）

```Bash

# 1. 创建 systemd 服务文件
sudo nano /etc/systemd/system/frpc.service
```

复制粘贴以下服务配置，无需修改任何内容：

```TOML

[Unit]
Description=Frp Client Service
After=network.target syslog.target
Wants=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
Restart=on-failure
RestartSec=5s
User=root

[Install]
WantedBy=multi-user.target
```

保存并退出编辑器后，执行以下命令启动服务：

```Bash

# 2. 重载 systemd 配置，识别新增服务
sudo systemctl daemon-reload

# 3. 启动 frpc 服务
sudo systemctl start frpc

# 4. 设置服务开机自启
sudo systemctl enable frpc

# 5. 验证服务状态（显示 active (running) 即部署成功）
sudo systemctl status frpc
```

---

## 四、穿透效果验证

### 4.1 SSH 内网服务器穿透验证

在任意可上网的电脑终端，执行以下命令，即可直接远程登录内网服务器：

```Bash

# 格式：ssh 内网服务器用户名@公网服务器IP -p 6000
ssh YourUserName@*** -p 6000
```

### 4.2 内网 WEB 服务穿透验证

在任意电脑的浏览器中，访问以下地址，即可打开内网部署的 WEB 服务/管理面板：

```Plain Text

http://你的公网服务器IP:8080
```

---

## 五、扩展：新增穿透规则方法

如需新增其他内网服务的穿透，仅需在内网服务器的 `/etc/frp/frpc.toml` 配置文件末尾，新增一个 `[[proxies]]` 配置块即可，模板如下：

```TOML

# 新增内网服务穿透模板
[[proxies]]
name = "自定义规则名称（不可重复）"
type = "tcp"
localIP = "127.0.0.1"
localPort = 内网服务实际端口
remotePort = 公网访问端口
```

修改完成后，执行以下命令重启客户端生效：

```Bash

sudo systemctl restart frpc
```

同时需在公网服务端的 `frps.toml` 配置文件的 `allowPorts` 中新增对应端口，并在云安全组放行该端口的TCP协议。
