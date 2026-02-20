---
title: "初始化你的Linux服务器"
date: 2026-02-19
description: "将一个裸的Linux系统打造成一个能用的开发环境"
tags: ['Linux', 'Ubuntu', 'Server', 'SSH', 'apt', 'Docker', 'pyenv']
series: ["自建Linux服务器实录"]
---

刚装好的Linux系统可能什么软件也没有，时区也不对，也没办法访问外网，下面让我们一个一个解决这些问题。

## 常用配置和依赖项安装（必备）

对于Ubuntu，你最好的软件管理方案就是apt，先来更新它一下

```bash
# 更新包索引+全量升级
sudo apt update 
sudo apt upgrade -y
# 清理无用残留包
sudo apt autoremove -y
# 内核更新后需重启生效
sudo reboot
```

然后是核心编译构建工具链，没了这些你可能啥也装不上。

```bash
# 包含gcc、g++、make、pkg-config等，源码编译、Python/node原生模块必备
sudo apt install build-essential cmake pkg-config -y
```

直接一套配齐

再然后是常用软件

```bash
sudo apt install -y curl wget vim htop net-tools dnsutils ncdu tree unzip zip
```

工具说明：
- `curl/wget`：文件下载、HTTP 接口调试
- `htop`：系统资源监控，比 top 更直观
- `dnsutils`：nslookup/dig，DNS 问题排查
- `ncdu`：磁盘占用分析，比 du 更易用
- `tree`：目录树结构展示

然后是git

```bash
sudo apt install git -y
# 配置全局Git信息
git config --global user.name "你的名称"
git config --global user.email "你的邮箱"
# 生成Git专用SSH密钥（和前面的SSH密钥可共用，也可单独生成）
ssh-keygen -t ed25519 -C "你的邮箱"
# 公钥路径：~/.ssh/id_ed25519.pub，添加到GitHub/GitLab等代码平台
```

最好的终端是zsh，用过的都说好
`zsh` + `oh-my-zsh` 替代默认 `bash`，自带命令补全、语法高亮、主题美化，大幅提升终端效率：
```bash
# 安装zsh
sudo apt install zsh -y
# 安装oh-my-zsh（官方一键脚本）
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# 切换默认shell为zsh
chsh -s $(which zsh)
```

安装必备插件
```bash
# 自动补全插件
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
# 语法高亮插件
git clone https://github.com/zsh-users/zsh-syntax-highlighting ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

安装后打开~/.zshrc，找到plugins那一行，改成
```bash
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

最后source一下让它生效
```bash
source ~/.zshrc
```

重新进入终端就可以了。

## Python 环境依赖（按需）

Python推荐使用Pyenv，这在需要多个python版本的时候很关键。Pyenv能让你在不同目录使用不同的python环境。

```bash
# 先安装pyenv编译Python所需的依赖（必装，否则编译会失败）
sudo apt install -y libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-openssl
# 安装pyenv（官方一键脚本）
curl https://pyenv.run | bash
```

然后添加环境变量到zsh（用bash的改为~/.bashrc）
```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc
# 生效配置
source ~/.zshrc
```

以下是Pyenv的常用命令
```bash
# 查看可安装的Python版本
pyenv install --list
# 安装指定版本（例如3.12 LTS）
pyenv install 3.12.7
# 设置全局默认Python版本
pyenv global 3.12.7
# 项目局部版本（进入项目目录执行，仅当前目录生效）
pyenv local 3.10.15
```

## Node.js/前端开发环境（nvm 多版本管理，按需）

安装
```bash
# 安装nvm（官方脚本，版本号可去nvm官网更新）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

添加环境变量到 zsh：
```bash
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.zshrc
echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.zshrc
echo '[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"' >> ~/.zshrc
# 生效配置
source ~/.zshrc
```

常用命令：
```bash
# 安装最新LTS版本
nvm install --lts
# 安装指定版本
nvm install 20.18.0
# 切换版本
nvm use 20.18.0
# 设置全局默认版本
nvm alias default 20.18.0
# 安装pnpm（高性能包管理器，替代npm/yarn）
npm install -g pnpm
```

## Docker 容器（按需但强推）

用容器隔离数据库、中间件、微服务，避免本地环境污染，保证开发 / 生产环境一致。

1. 安装 Docker 与 Docker Compose（官方最新版）
```bash
# 安装依赖
sudo apt install -y ca-certificates curl gnupg lsb-release
# 添加Docker官方GPG密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
# 添加Docker官方源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# 安装Docker引擎与Compose插件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

配置免 sudo 执行 docker（必须重新登录用户生效）：
```bash
sudo usermod -aG docker $USER
# 临时生效（无需重启）
newgrp docker
# 验证安装
docker run hello-world
docker compose version
```

2. 配置国内 Docker 镜像源（提升拉取速度）
```bash
sudo nano /etc/docker/daemon.json
```

添加以下内容：
```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

重启 Docker 生效：
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 时区与语言同步（必备）

安装好后默认时区是零时区，需要改成自己的时区

```bash
# 设置时区为上海
sudo timedatectl set-timezone Asia/Shanghai
# 启用系统时间同步
sudo timedatectl set-ntp true
# 查看状态
timedatectl status
```

Locale 配置
```bash
# 生成缺失的 Locale
sudo locale-gen en_US.UTF-8
# 编辑 /etc/default/locale 文件
sudo nano /etc/default/locale
```
在locale文件中添加或者修改为：
```
LANG=en_US.UTF-8
LC_ALL=en_US.UTF-8
LANGUAGE=en_US:en
```

## 解除100G限制 (必备)

Ubuntu Server 默认安装时，会把整个磁盘的可用空间划入卷组 (VG)，但仅给根逻辑卷 (LV) 分配 100G，剩余空间在卷组中闲置。

说人话就是不管你有多大的盘你都只能用100G，如果你不设置一下剩下的空间都会被闲置。

直接按以下步骤操作即可。

1. 确认卷组空闲空间：
```bash
sudo vgs
```
查看输出中`Free`列，有数值即存在可分配空闲空间。

2. 扩展根逻辑卷（占用全部空闲空间）：
```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```
说明：`-l +100%FREE`表示将卷组所有剩余空间全部分配给根逻辑卷，无需手动计算大小。

3. 同步扩容文件系统（让系统识别新空间）：

```bash
# 先确认文件系统类型：
df -T /
# ext4格式（Ubuntu默认）：
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
# 如果是xfs格式，那么
# sudo xfs_growfs /
```

4. 验证扩容结果：
```bash
df -h /
```

## 解决ssh连接时远程自动关闭连接问题（按需但强推）

新服务器一般你远端连接之后几分钟内就会断开，这是因为服务端设置的问题

1. 备份SSH配置文件（必做，防止配置错误）

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

2. 编辑SSH服务配置文件

```bash
sudo nano /etc/ssh/sshd_config  # 新手推荐nano，也可替换为vim
```
在文件末尾添加/修改以下参数（取消原有相关参数注释并替换）：

```ini
# 服务端每60秒发送一次心跳包，维持连接活性
ClientAliveInterval 60
# 设为0 = 无限次发送心跳，永不主动断开（核心参数）
ClientAliveCountMax 0
# 开启TCP层保活，适配网络设备
TCPKeepAlive yes
```

3. 校验配置并重启服务生效

```bash
# 校验配置文件语法（无输出即无错误）
sudo sshd -t
# 重启SSH服务，配置永久生效
sudo systemctl restart sshd
# 查看生效参数
sudo sshd -T | grep -E 'clientalive|tcpkeepalive'
```

正常输出应为：clientaliveinterval 60、clientalivecountmax 0、tcpkeepalive yes

## Clash代理配置（按需）

如果你没有一个开箱即用的proxy，或者习惯了用自己电脑上的clash，可以这样添加代理

[Clash for linux server](https://www.igisv.com/ubuntu-server-clash-wordpress/)


---

完成了以上这些，你的 Linux 服务器才可以说是比较好用了，然而，如果想随时随地从外网访问，但自己家里又没有公网IP怎么办呢？没关系，下一篇将介绍开源的Frp内网穿透方案，只要你有一个能公网访问的服务器，就可以直接访问你内网的机器。