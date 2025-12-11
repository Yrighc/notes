# 防火墙命令行控制

本文档提供了在不同操作系统上使用命令行管理防火墙的常用命令。

## Linux

Linux 系统有多种防火墙管理工具。不同发行版默认的工具不同，本节按发行版分类介绍。

### Ubuntu / Debian (使用 UFW)

Ubuntu 和 Debian 默认使用 `ufw` (Uncomplicated Firewall)，它是一个旨在简化 `iptables` 操作的用户友好前端。

**1. 启动与停止**
```bash
# 启用防火墙并设置开机自启
sudo ufw enable

# 禁用防火墙
sudo ufw disable
```

**2. 状态查看**
```bash
# 查看基本状态
sudo ufw status

# 查看带编号的规则列表
sudo ufw status numbered
```

**3. 增删端口策略**
```bash
# 允许外部访问 22/tcp 端口 (SSH)
sudo ufw allow 22/tcp

# 允许外部访问 80/tcp 端口 (HTTP)
sudo ufw allow http

# 拒绝外部访问 8080 端口
sudo ufw deny 8080/tcp

# 删除第一条规则 (先用 status numbered 查看编号)
sudo ufw delete 1
```

**4. 重新加载**
`ufw` 在每次修改规则后会自动生效，通常不需要手动重载。如果需要重置所有规则并重新加载配置文件，可以使用：
```bash
# 重置防火墙到安装时的默认状态
sudo ufw reset
# 然后重新启用
sudo ufw enable
```

### CentOS / RHEL / Fedora (使用 firewalld)

CentOS, RHEL, Fedora 等发行版默认使用 `firewalld`，它支持网络“区域”(zones) 的概念，管理更灵活。

**1. 启动与停止**
```bash
# 启动 firewalld 服务
sudo systemctl start firewalld
# 设置 firewalld 开机自启
sudo systemctl enable firewalld

# 停止 firewalld 服务
sudo systemctl stop firewalld
# 禁止 firewalld 开机自启
sudo systemctl disable firewalld
```

**2. 状态查看**
```bash
# 查看 firewalld 运行状态
sudo firewall-cmd --state

# 查看当前区域(默认为 public)的所有配置
sudo firewall-cmd --list-all
```

**3. 增删端口策略**
`firewalld` 的规则分为“临时”和“永久”。临时规则立即生效，但重启后失效。永久规则需要重载后才生效。

```bash
# 永久开放 8080/tcp 端口
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent

# 永久关闭 8080/tcp 端口
sudo firewall-cmd --zone=public --remove-port=8080/tcp --permanent

# 永久开放 http 服务 (即 80/tcp)
sudo firewall-cmd --zone=public --add-service=http --permanent
```

**4. 重新加载配置**
在修改了永久规则后，必须重新加载配置才能生效。
```bash
sudo firewall-cmd --reload
```

### 通用工具: iptables

`iptables` 是一个功能强大但更复杂的底层工具，适用于所有 Linux 发行版。

**1. 状态查看**
```bash
# 以详细、数字化的格式列出所有规则
sudo iptables -L -v -n
```

**2. 增删端口策略**
```bash
# 允许入站的 SSH 连接
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 拒绝来自特定 IP 的所有入站连接
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# 删除规则 (需要先找到规则内容和位置)
# 例如，删除上面添加的 SSH 规则
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT
```

**3. 启动与停止/重载**
`iptables` 本身没有启动停止的概念，它是对内核规则的直接修改。规则的持久化依赖于特定发行版的工具。

*   **Debian/Ubuntu:**
    ```bash
    # 安装工具
    sudo apt-get install iptables-persistent
    # 保存当前规则
    sudo netfilter-persistent save
    ```
*   **CentOS/RHEL:**
    ```bash
    # 保存当前规则
    sudo service iptables save
    ```
    重载通常意味着清空所有规则链 (`iptables -F`) 然后从保存的文件中重新加载。

## Windows

Windows 使用 `netsh advfirewall` 来通过命令行管理防火墙。

**1. 检查状态**
```powershell
netsh advfirewall show allprofiles state
```

**2. 启用/禁用防火墙**
```powershell
# 启用
netsh advfirewall set allprofiles state on

# 禁用
netsh advfirewall set allprofiles state off
```

**3. 添加入站/出站规则**
```powershell
# 添加入站规则以允许端口 8080
netsh advfirewall firewall add rule name="Allow Port 8080" dir=in action=allow protocol=TCP localport=8080

# 添加出站规则以阻止到特定 IP 的连接
netsh advfirewall firewall add rule name="Block IP" dir=out action=block remoteip=1.2.3.4
```

**4. 删除规则**
```powershell
netsh advfirewall firewall delete rule name="Allow Port 8080"
```

**5. 重置防火墙**
```powershell
netsh advfirewall reset
```

## macOS

macOS 使用 `pf` (Packet Filter) 作为其防火墙后端。`pfctl` 是管理 `pf` 的命令行工具。

**1. 检查状态**
```bash
sudo pfctl -s info
```

**2. 启用/禁用防火墙**
```bash
# 启用
sudo pfctl -e

# 禁用
sudo pfctl -d
```

**3. 管理规则**
macOS 的防火墙规则在 `/etc/pf.conf` 文件中定义。

*   **编辑规则文件:**
    ```bash
    sudo nano /etc/pf.conf
    ```
    示例规则 (允许 SSH 和 HTTP):
    ```
    # 允许所有出站流量
    pass out on en0 all

    # 阻止所有入站流量
    block in on en0 all

    # 允许入站 SSH 和 HTTP
    pass in on en0 proto tcp from any to any port {22, 80}
    ```

*   **测试配置文件语法:**
    ```bash
    sudo pfctl -nf /etc/pf.conf
    ```

*   **加载规则:**
    ```bash
    sudo pfctl -f /etc/pf.conf
    ```

**4. 查看当前加载的规则**
```bash
sudo pfctl -s rules
```
