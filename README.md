# 在 Debian 12 VPS 上安装和配置 systemd-resolved 以使用 DNS（DoH/DoT）

本文档整理了在 Debian 12 VPS 上安装 systemd-resolved，配置 DNS over HTTPS (DoH) 或 DNS over TLS (DoT)，并处理相关端口问题（443、853、5355）的完整步骤。目标是确保安全、高效的 DNS 解析，同时避免公网暴露（如 5355 端口的 LLMNR）。内容基于你之前的提问（配置 DoH 遇到 443 端口冲突、切换到 DoT、关注 5355 端口等）。

## 背景
- 目标：在 Debian 12 VPS 上配置 systemd-resolved 使用 Cloudflare（1.1.1.1, 1.0.0.1）和 Google（8.8.8.8, 8.8.4.4）的 DNS 服务器，支持 IPv4 和 IPv6 双栈，优先使用 DoH（443 端口）或 DoT（853 端口）。
- 问题：
  - 初始尝试配置 DoH，疑似 443 端口冲突导致未生效。
  - 切换到 DoT（DNSOverTLS=yes），但需确认配置正确性。
  - 发现 systemd-resolved 监听 0.0.0.0:5355（LLMNR），可能对公网开放。
- 环境：Debian 12（systemd 252 或更高，支持 DoH 和 DoT）。

## 步骤 1：安装 systemd-resolved
1. 检查是否已安装：
   检查 systemd-resolved 是否存在：
   ```bash
   dpkg -l | grep systemd-resolved
   ```
   - 如果无输出，说明未安装。

2. **安装 systemd-resolved**：
   ```bash
   sudo apt update
   sudo apt install systemd-resolved
   ```
启用并启动服务服务**：
   ```bash
   sudo systemctl enable --now systemd-resolved
   ```
验证服务状态状态**：
   ```bash
   systemctl status systemd-resolved
   ```
   - 确保输出显示 active (running)。

5. **配置 /etc/resolv.conf**：
   确保 /etc/resolv.conf 指向 systemd-resolved 的 stub 解析器：
   ```bash
   ls -l /etc/resolv.conf
   ```
   - 应显示为符号链接：/etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf。
   - 检查内容：
     ```bash
     cat /etc/resolv.conf
     ```
     - 预期：
      ```bash
       nameserver 127.0.0.53
      ```
   - 如果不是符号链接，手动修复：
     ```bash
     sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
     ```

## 步骤 2：配置 DNS over HTTPS (DoH)
DoH 使用 443 端口，适合伪装为 HTTPS 流量，但需确保 443 端口未被检查 443 端口占用3 端口占用**：
   ```bash
   sudo ss -tuln | grep :443
   ```
   - 如果有输出（如 0.0.0.0:443），可能是 Nginx、Apache 等占用。
   - 释放端口（以 Nginx 为例）：
     ```bash
     sudo nano /etc/nginx/sites-available/default
     ```
     - 修改 listen 443; 为 listen 8443;。
     - 重启：
       ```bash
       sudo systemctl restart nginx
       ```

2. **配置 /etc/systemd/resolved.conf**：
   ```bash
   sudo nano /etc/systemd/resolved.conf
   ```
   - 添加以下内容（推荐简单配置）：
     ```bash
     [Resolve]
     DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
     DNSOverHTTPS=yes
     LLMNR=no
     ```
     - **说明**：
       - DNS：Cloudflare 和 Google 的 IPv4 地址，支持 DoH。
       - DNSOverHTTPS=yes：启用 DoH，systemd-resolved 自动识别服务器。
       - LLMNR=no：禁用 LLMNR，关闭 5355 端口（见步骤 5）。
     - 如果需要 IPv6：
       ```bash
       [Resolve]
       DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 2606:4700:4700::1111 2606:4700:4700::1001 2001:4860:4860::8888 2001:4860:4860::8844
       DNSOverHTTPS=yes
       LLMNR=no
       ```
3. **重启服务**：
   ```bash
   sudo systemctl restart system验证 DoH
   ```

4. **验证 DoH**：
   - 检查状态：
     ```bash
     resolvectl status
     ```
     - 确认显示 DNS Protocol: DoH 和正确的 DNS 服务器。
   - 测试解析：
     ```bash
     dig google.com
     ```
     - 确保通过 127.0.0.53 解析。
   - 验证 DoH 流量（443 端口）：
     ```bash
     sudo tcpdump -i any port 443
     ```
     - 在另一终端运行：
       ```bash
       resolvectl query google.com
       ```
     - 观察是否有指向 1.1.1.1 或 8.8.8.8 的 HTTPS 流量。

5. **确保 443 端口开放**：
   ```bash
   sudo ufw allow out 443
   ```

## 步骤 3：配置 DNS over TLS (DoT)（备选方案）
如果 443 端口冲突无法解决，DoT 使用 853 端口，是更可靠的选择。

1. **配置 /etc/systemd/resolved.conf**：
   ```bash
   sudo nano /etc/systemd/resolved.conf
   ```
   - 添加：
     ```bash
     [Resolve]
     DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
     DNSOverTLS=yes
     LLMNR=no
     ```
     - **说明**：
       - DNSOverTLS=yes：启用 DoT，使用 853 端口。
       - 域名（如 #cloudflare-dns.com）非必须，IP 地址足够。
     - 可选 IPv6：
       ```bash
       [Resolve]
       DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 2606:4700:4700::1111 2606:4700:4700::1001 2001:4860:4860::8888 2001:4860:4860::8844
       DNSOverTLS=yes
       LLMNR=no
       ```
2. **重启服务**：
   ```bash
   sudo systemct验证 DoTsystemd-resolved
   ```

3. **验证 DoT**：
- 检查状态：
     ```bash
     resolvectl status
     ```
     - 确认显示 Protocols: +DNSOverTLS。
   - 测试解析：
     ```bash
     dig ```1.1.1.1 google.com
     ```
   - 验证 DoT 流量（853 端口）：
     ```bash
     sudo tcpdump -i any port 853
     ```
     - 在另一终端运行：
       ```bash
       resolvectl query google.com
       ```

4. 确保 853 端口开放：
   ```bash
   sudo ufw allow out 853
   ```

## 步骤 4：处理 5355 端口（LLMNR）
你注意到 systemd-resolved 监听 0.0.0.0:5355，可能对公网开放。LLMNR 用于局域网主机名解析，VPS 通常无需启用。

1. 禁用 LLMNR（推荐）：
   - 编辑 /etc/systemd/resolved.conf：
     ```bash
     sudo nano /etc/systemd/resolved.conf
     ```
     - 确保包含：
      ```bash
       LLMNR=no
      ```
   - 重启：
     ```bash
     sudo systemctl restart systemd-resolved
     ```
   - 验证：
     ```bash
     sudo ss -tuln | grep :5355
     ```
     - 无输出表示 5355 端口已关闭。
     ```bash
     resolvectl status
     ```
     - 确认 Protocols: -LLMNR。

2. 限制为局域网（如果作为网关）：
   如果 VPS 是网关，为 192.168.x.x 提供 LLMNR：
   ```bash
   sudo ufw allow from 192.168.0.0/16 to any port 5355
   sudo ufw deny 5355
   ```
   - 验证：
     ```bash
     sudo ufw status
     ```
   - 测试公网访问：
     ```bash
     nmap -p 5355 <VPS公网IP> -sU
     ```
     - 应显示 blocked 或 filtered。

## 步骤 5：验证 53 端口
systemd-resolved 默认在 127.0.0.53:53 提供 stub 解析器，仅限本地。

1. 检查监听：
   ```bash
   sudo ss -tuln | grep :53
   ```
   - 预期：127.0.0.53:53。
   - 如果显示 0.0.0.0:53，检查其他 DNS 服务（如 bind9, unbound）：
     ```bash
     systemctl list-units --type=service | grep -E 'dns|bind|unbound'
     ```

2. 限制其他 DNS 服务（如有）：
   - 对于 unbound：
     ```bash
     sudo nano /etc/unbound/unbound.conf
     ```
     - 设置：
       ```bash
       server:
           interface: 127.0.0.1
       ```
     - 重启：
       ```bash
       sudo systemctl restart unbound
       ```

## 步骤 6：处理 NetworkManager 冲突
如果使用 NetworkManager，确保不覆盖 DNS：
```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```
- 添加：
  ```bash
  [main]
    dns=systemd-resolved
  ```
- 重启：
```bash
sudo systemctl restart NetworkManager
```

## 步骤 7：是否配置过多
- 当前配置：使用 Cloudflare 和 Google 的 IPv4（4 个地址）或 IPv4+IPv6（8 个地址）。
- 分析：
- 8 个地址合理，systemd-resolved 会选择最快响应。
- 如果网络不支持 IPv6（测试：ping6 google.com），可精简：
  ```bash
  [Resolve]
  DNS=1.1.1.1 1.0.0.1
  DNSOverTLS=yes
  LLMNR=no
  ```
- 测试性能：
```bash
dig ```1.1.1.1 google.com
dig ```8.8.8.8 google.com
```

## 总结
- DoH 配置：优先尝试（需 443 端口可用）：
  ```bash
  [Resolve]
  DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
  DNSOverHTTPS=yes
  LLMNR=no
  ```
- DoT 配置：若 443 端口冲突，使用：
  ```bash
  [Resolve]
  DNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
  DNSOverTLS=yes
  LLMNR=no
  ```
- 端口：
- DoH：开放出站 443 端口。
- DoT：开放出站 853 端口。
- 5355：禁用 LLMNR 或限制为 192.168.x.x。
- 53：确保 127.0.0.53:53（仅本地）。
- 验证：使用 resolvectl status, dig, tcpdump 确认 DoH/DoT 和端口状态。
