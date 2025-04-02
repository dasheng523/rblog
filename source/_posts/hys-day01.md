---
title: hys/day01
date: 2025-04-02 13:10:15
categories:
    - diary
tags:
    - author:Yesheng
---
## 前言

这几年我总是有意无意地弄虚拟机，一直卡在 bios 虚拟化上面，甚至以为主板是很老旧了，需要更新电脑。早上还都 JD 看了一圈主板，后来想想觉得还是再试一遍吧。于是我仔仔细细又检查了一遍 BIOS，把所有跟虚拟化的全勾上，居然好了。这下终于正常了，开心。

接下来我看看如何弄技术日志吧，这是作业的一部分。可以在这里看到博客相关的作业：https://github.com/rcore-os/blog

然而虚拟机还需要配置很多东西，特别是虚拟机路径不要放在 C 盘中，咨询 AI 后，也找到了方法。


## 虚拟机挪位置

1. **停止 WSL 服务**（确保所有分发版关闭）：

    ```powershell
    wsl --shutdown
    ```
2. **导出当前 Ubuntu 系统**（备份为 tar 文件）：

    ```powershell
    wsl --export Ubuntu D:\ubuntu_backup.tar
    ```

    * `Ubuntu`：分发版名称（可通过 `wsl -l` 查看）
    * `D:\ubuntu_backup.tar`：导出的备份文件路径（可自定义盘符）
3. **注销原 Ubuntu 系统**：

    ```powershell
    wsl --unregister Ubuntu
    ```
4. **导入 Ubuntu 到目标磁盘**（例如 D 盘）：

    ```powershell
    wsl --import Ubuntu D:\wsl-ubuntu\ D:\ubuntu_backup.tar --version 2
    ```

    * `D:\wsl-ubuntu\`：新安装目录（需提前创建）
    * `--version 2`：强制使用 WSL2
5. 验证安装路径

    在 PowerShell 中运行以下命令，查看 Ubuntu 的存储路径：

    ```powershell
    Get-ChildItem HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss\ |
    ForEach-Object { (Get-ItemProperty $_.PSPATH) } |
    Select-Object DistributionName, BasePath
    ```


## 配置代理

### 一、获取宿主机的 IP 地址

WSL2 中需使用宿主机的 **虚拟网络 IP**，而非 `localhost` 或 `127.0.0.1`。
通过以下命令获取宿主机 IP：

```bash
cat /etc/resolv.conf | grep nameserver | awk '{print $2}'
```

或使用 PowerShell 查询：

```powershell
(Get-NetIPAddress -InterfaceAlias "vEthernet (WSL)" -AddressFamily IPv4).IPAddress
```

通常 IP 格式为 `172.x.x.1`。

---

### 二、WSL2 中设置代理环境变量

1. **临时生效（当前终端会话）** ：

    ```bash
    export host_ip=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
    export http_proxy="http://$host_ip:7890"
    export https_proxy="http://$host_ip:7890"
    export all_proxy="socks5://$host_ip:7891"  # 若使用 SOCKS5 代理
    ```
2. **永久生效（写入 Shell 配置文件）** ：

    ```bash
    echo "export host_ip=\$(cat /etc/resolv.conf | grep nameserver | awk '{print \$2}')" >> ~/.bashrc
    echo "export http_proxy=\"http://\$host_ip:7890\"" >> ~/.bashrc
    echo "export https_proxy=\"http://\$host_ip:7890\"" >> ~/.bashrc
    echo "export all_proxy=\"socks5://\$host_ip:7891\"" >> ~/.bashrc  # 可选
    source ~/.bashrc  # 立即生效
    ```

### 三、宿主机**放行多个端口**（例如 HTTP 7890 和 SOCKS5 7891）：

```powershell
New-NetFirewallRule `
    -DisplayName "Allow WSL Proxy TCP 7890-7891" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 7890,7891
```

#### **验证规则是否生效**：

```powershell
Get-NetFirewallRule -DisplayName "Allow WSL Proxy*" | Format-Table DisplayName,Enabled
```

输出中 `Enabled` 为 `True` 表示规则已启用。


### 检查代理软件状态

1. **确保代理工具正在运行**：

    * 检查宿主机上的代理软件（如 Clash、V2Ray）是否已启动，并确认监听的端口（如 `7890`）正确。
    * **Clash 用户**：确认开启了 **Allow LAN**（允许局域网连接）选项。


### **验证代理端口是否可达**

在 WSL2 的 Ubuntu 终端中，使用 `telnet` 测试端口连通性：

```bash
host_ip=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
telnet $host_ip 7890  # 按 Ctrl+] 退出
```
## 安装开发环境

### git

**步骤：**

1. 添加 Git 官方维护的 PPA 仓库：

    ```bash
    sudo add-apt-repository ppa:git-core/ppa
    ```
2. 更新软件包列表并安装 Git：

    ```bash
    sudo apt update
    sudo apt install git
    ```

### Nodejs

* **安装步骤**：

  ```bash
  curl -fsSL https://fnm.vercel.app/install | bash
  ```
* **常用命令**：

  ```bash
  fnm install v22.14.0
  ```

### Java

**安装OpenJDK**

```bash
# 安装OpenJDK 17（当前主流LTS版本）
sudo apt install openjdk-17-jdk

# 或指定其他版本（如JDK 8/11）
sudo apt install openjdk-8-jdk openjdk-11-jdk
```

通过官方仓库安装，自动配置环境变量

**验证安装**

```bash
java -version  # 显示类似"openjdk 17.0.1"即成功
```

### Rust

用的是 https://rcore-os.cn/arceos-tutorial-book/ch01-02.html 这里的安装教程，安装了`nightly`版本。也配置了镜像地址。

并安装了一些必要的软件包。


### OpenSSH

```bash
sudo apt update
sudo apt install openssh-server
```

* 修改 SSH 配置文件：

  ```bash
  sudo vi /etc/ssh/sshd_config
  ```
* 关键参数修改：

  ```text
  Port 22                 # 默认端口（可自定义如 2222）
  ListenAddress 0.0.0.0   # 监听所有网络接口
  PermitRootLogin yes     # 允许 root 登录（可选）
  PasswordAuthentication yes  # 启用密码验证
  ```

`sudo service ssh restart`

**获取 WSL2 的 IP 地址**（在 Ubuntu 中运行）：

```bash
ip addr show eth0 | grep inet
```


## 编写博客

* 先迁出代码   git@github.com:dasheng523/rblog.git
* 安装hexo

```bash
npm install hexo-cli -g 
hexo n "<blog-title>"
```
