# 防火墙命令行控制

本文档提供了在不同操作系统上使用命令行管理防火墙的常用命令。

## Linux

Linux 系统有多种防火墙管理工具，这里介绍三种常见的工具：`ufw`，`firewalld` 和 `iptables`。

### UFW (Uncomplicated Firewall) - (Debian/Ubuntu)

UFW 是一个旨在简化 `iptables` 操作的用户友好前端。

**1. 检查状态**
```bash
sudo ufw status
```

**2. 启用/禁用防火墙**
```bash
# 启用
sudo ufw enable

# 禁用
sudo ufw disable
```

**3. 设置默认策略**
```bash
# 拒绝所有传入连接
sudo ufw default deny incoming

# 允许所有传出连接
sudo ufw default allow outgoing
```

**4. 允许/拒绝连接**
```bash
# 允许特定端口 (例如：SSH)
sudo ufw allow 22/tcp

# 允许特定服务 (例如：HTTP)
sudo ufw allow http

# 允许来自特定 IP 地址的连接
sudo ufw allow from 192.168.1.100

# 拒绝特定端口
sudo ufw deny 8080/tcp
```

**5. 删除规则**
```bash
# 列出规则并显示编号
sudo ufw status numbered

# 根据编号删除规则
sudo ufw delete 1
```

**6. 重置防火墙**
```bash
sudo ufw reset
```

### firewalld - (RHEL/CentOS/Fedora)

`firewalld` 是一个动态管理的防火墙，支持网络“区域”(zones) 的概念。

**1. 检查状态**
```bash
sudo firewall-cmd --state
```

**2. 启用/禁用防火墙**
```bash
# 启动并设置为开机自启
sudo systemctl start firewalld
sudo systemctl enable firewalld

# 停止并禁用开机自启
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

**3. 管理区域和规则**
```bash
# 查看当前区域和规则
sudo firewall-cmd --list-all

# 允许服务 (临时)
sudo firewall-cmd --zone=public --add-service=http

# 允许服务 (永久)
sudo firewall-cmd --zone=public --add-service=http --permanent

# 允许端口 (永久)
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
```

**4. 重新加载配置**
在添加或修改永久规则后，需要重新加载防火墙以使更改生效。
```bash
sudo firewall-cmd --reload
```

### iptables

`iptables` 是一个功能强大但更复杂的工具，用于配置 Linux 内核防火墙。

**1. 列出规则**
```bash
sudo iptables -L -v -n
```

**2. 允许/拒绝连接**
```bash
# 允许 SSH 连接
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 拒绝来自特定 IP 的连接
sudo iptables -A INPUT -s 192.168.1.100 -j DROP
```

**3. 保存规则**
`iptables` 规则在重启后会丢失。保存规则的方法因发行版而异。

*   **Debian/Ubuntu:**
    ```bash
    sudo apt-get install iptables-persistent
    sudo netfilter-persistent save
    ```
*   **CentOS/RHEL:**
    ```bash
    sudo service iptables save
    ```

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
