# Linux 运维实战手册：Web 服务部署、监控与自动化运维

> 本手册基于企业级 Web 服务部署真实项目整理。涵盖 CentOS 7 系统初始化、Nginx 服务部署、Shell 自动化巡检以及故障排查的完整运维闭环，可供日常查阅与参考。

## 📖 目录
1. [环境准备与系统安装](#1-环境准备与系统安装)
2. [服务器基础初始化配置](#2-服务器基础初始化配置)
3. [Nginx Web 服务部署与配置](#3-nginx-web-服务部署与配置)
4. [Nginx 日志管理与故障排查](#4-nginx-日志管理与故障排查)
5. [服务器基础监控与日常巡检](#5-服务器基础监控与日常巡检)
6. [运维经验与最佳实践总结](#6-运维经验与最佳实践总结)

---

## 1. 环境准备与系统安装
### 1.1 虚拟机环境搭建
- **虚拟化软件**：VMware Workstation
- **操作系统**：CentOS 7.x (建议使用 Minimal 最小化安装)
- **硬件配置建议**：
  - 最低：1核CPU / 1GB内存 / 20GB硬盘
  - 推荐：2核CPU / 4GB内存 / 40GB+硬盘

### 1.2 系统基本配置
- **磁盘分区**：推荐自定义分区，`/boot` 分配 1GB (ext4)，`swap` 根据内存大小配置（内存<4GB设2倍，>4GB设4~8GB），根目录 `/` 分配剩余空间 (xfs)。
- **网络与主机名**：开启网卡，配置静态 IP（`BOOTPROTO=static`），自定义主机名（如 `centos7-server`）。

---

## 2. 服务器基础初始化配置
> 操作前请确保已获取 `root` 权限。

### 2.1 设置静态 IP
1. 查看网络设备名称：`ip addr`
2. 修改网卡配置文件：`vi /etc/sysconfig/network-scripts/ifcfg-eth0` (或 `ens33`)
3. 修改参数：
    ```bash
    BOOTPROTO=static
    IPADDR=192.168.1.xxx   # 根据实际网段设置
    NETMASK=255.255.255.0
    GATEWAY=192.168.1.1    # 网关
    DNS1=8.8.8.8           # 建议写公共DNS
    ```
4. 重启网络服务：`systemctl restart network`
5. 验证网络连通性：`ping 192.168.1.1 -c 4` 和 `ping www.baidu.com -c 4`

### 2.2 防火墙配置 (测试环境建议关闭)
1. 临时关闭：`systemctl stop firewalld`
2. 永久关闭：`systemctl disable firewalld`
3. 验证状态：`systemctl status firewalld`
> *注：生产环境建议仅开放必要的业务端口*

### 2.3 SELinux 配置 (测试环境建议关闭)
1. 临时关闭：`setenforce 0`
2. 永久关闭：`vi /etc/selinux/config`，修改 `SELINUX=disabled`。
3. 验证状态：`getenforce`，重启服务器后查看 `sestatus`。

### 2.4 调整时区与时间同步
1. 查看当前时区：`timedatectl`
2. 设置时区为上海：`timedatectl set-timezone Asia/Shanghai`
3. 安装并配置时间同步服务 (Chrony)：
    ```bash
    yum install -y chrony
    vi /etc/chrony.conf    # 修改为国内 NTP 服务器，例如：server ntp.aliyun.com iburst
    systemctl start chronyd
    systemctl enable chronyd```
4. 验证时间同步效果：`chronyc sources -v` 和 `date`

### 2.5 优化系统资源限制（文件打开数）
1. 查看当前限制：`ulimit -n`
2. 临时调整：`ulimit -n 65535`
3. 永久调整：编辑` vi /etc/security/limits.conf`，在文件末追加：
    ```bash
    * soft nofile 65535
    * hard nofile 65535```
    重启服务器或重新登录终端生效。

### 2.6 内核参数优化 (sysctl)
1. 编辑内核配置文件：`vi /etc/sysctl.conf`
2. 添加常用内核调优参数（建议根据业务调整）：
    ```bash
    net.core.somaxconn = 65535    # 网络并发优化
    net.core.rmem_max = 65535
    net.ipv4.tcp_max_tw_buckets = 65535
    vm.swappiness = 10    # 内存优化（优先使用物理内存，避免过度使用交换分区）
    fs.file-max = 655350    # 文件系统优化
    ```
3.使配置生效（无需重启）：`sysctl -p`

### 2.7 更新软件源及清理多余软件
1. 备份并更换为阿里云 YUM 源（可根据环境选择）：
    ```bash
    curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
    yum makecache
    ```
2. 更新系统软件：`yum update -y`
3. 清理无用软件（清理游戏、打印机、蓝牙等组件）：
    ```bash
    yum remove -y games postfix cups ...
    ```
---

## 3. Nginx Web 服务部署与配置
### 3.1 使用 YUM 安装 Nginx
1. 配置 Nginx 官方源仓库：
    ```bash
    vi /etc/yum.repos.d/nginx.repo
    ```
    写入内容：
    ```ini
    [nginx-stable]
    name=nginx stable repo
    baseurl=http://nginx.org/packages/centos/7/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    ```
2. 导入 GPG 密钥并清理缓存：
    `rpm --import https://nginx.org/keys/nginx_signing.key`和 `yum makecache fast`
3. 执行安装：`yum install -y nginx`
### 3.2 Nginx 核心配置与启动
1. 编辑主配置文件：`vi /etc/nginx/nginx.conf`
2. 自定义端口和根目录示例（在 `server` 块中修改）：
    ```nginx
    server {
        listen 8080;            # 自定义监听端口
        server_name localhost;
        root /data/www;         # 自定义网站根目录
        index index.html index.htm;
        location / {
            autoindex on;
        }
    }```
3. 创建目录并设置权限：`mkdir -p /data/www` 和 `chown -R nginx:nginx /data/www`
4. **执行配置预检查（非常重要）**：`nginx -t`（若返回 success 则说明配置正确）
5. 启动并设置开机自启：
    ```bash
    systemctl start nginx
    systemctl enable nginx```
6. 状态检查与进程验证：
    ```bash
    systemctl status nginx
    ps aux | grep nginx
    ss -tnlp | grep nginx   # 查看端口监听情况```

---

## 4. Nginx 日志管理与故障排查
### 4.1 基础命令实时查看日志
- 查看访问日志：`tail -f /var/log/nginx/access.log` (跟踪输出)
- 查看错误日志：`tail -f /var/log/nginx/error.log` (服务报错时查阅)
- 进阶：过滤 404 错误：`tail -f /var/log/nginx/access.log | grep " 404 "`
### 4.2 排查 404 错误的常用流程
1. 提取请求路径：从日志中获取请求的文件，例如` /static/css/main.css`。
2. 定位网站根目录：通过 `cat /etc/nginx/nginx.conf | grep root` 获取配置文件中的 `root `地址。
3. 核实文件路径与权限：
    ```bash
    ls -l /root实际路径/对应请求路径   
    # 示例：ls -l /usr/share/nginx/html/static/css/main.css```
    *常见原因：文件未上传、路径拼写大小写错误（Linux 严格区分大小写）、Nginx 用户（`nginx`）读取权限不足。*
### 4.3 常用日志处理技巧
- 退出实时追踪：按 `Ctrl + C`
- 查看日志最后 N 行：`tail -n 50 /var/log/nginx/access.log
- 过滤特定 IP 的访问记录：`grep "192.168.1.101" /var/log/nginx/access.log`
- 分析统计特定 IP 的访问次数：`grep "192.168.1.101" /var/log/nginx/access.log | wc -l`
- 处理压缩过的历史日志：`zgrep "192.168.1.101" /var/log/nginx/access.log.20251222.gz`

---

## 5. 服务器基础监控与日常巡检
> 每日按此流程巡检，实现“监控检查-问题处理-配置保障”的闭环。
### 5.1 资源监控检查 (核心)
**1. CPU 与内存监控：**`top`
- 监控要点：关注 `%Cpu(s)` 中的 `us` (用户) 和 `sy`(系统)，及 `Mem` (物理内存) 和 `Swap` 使用率。
- **异常判断与处理标准**：
  - **CPU占用（us+sy）**：正常 `≤ 70%`；异常 `> 85%`。
    - 解决：按 `P` 键按 CPU 排序，找到高占用进程确认是否合法，异常则 `kill -9 [PID]`。
  - **物理内存使用率**：正常 `≤ 80%`；异常 `> 90%`。
    - 解决：按 M 键排序，找到异常进程清理内存缓存（`echo 3 > /proc/sys/vm/drop_caches`）。
  - **Swap 使用率**：正常 `≤ 20%`；异常` > 50%`。
    - 解决：说明内存严重不足，需扩容物理内存。
**2. 磁盘空间监控：**`df -h`
- **异常判断**：分区使用率正常 `≤ 80%`；异常`> 85%`。
- **排查手段**：使用 `du -sh /目录` 查找大文件，优先清理过期的日志、临时文件或缓存。
### 5.2 核心服务状态检查
1. 检查 Nginx 运行状态：`systemctl status nginx`，若显示` inactive `则需启动，显示 `failed` 需查看日志` journalctl -xe | grep nginx`。
2. 验证开机自启状态：`systemctl is-enabled nginx`，若输出 `disabled` 需执行 `systemctl enable nginx`。
### 5.3 运维归档与记录
日常巡检结束后，需记录以下核心内容：
- 基础信息：日期、服务器 IP、主机名。
- 监控数据：CPU、内存、磁盘峰值及平均值。
- 问题记录：发现的异常、原因分析、处理措施及结果。
- 优化建议：针对反复出现的问题，提出脚本自动清理或扩容的长期方案。

---

## 6. 运维经验与最佳实践总结
### 6.1 权限管理：遵循“最小权限原则”
- 日常操作尽量使用普通用户，仅在需重启服务或修改配置时才切换 `root` (`su - root`)，避免误操作导致系统崩溃。
- 为新起服务建立独立用户（如 Nginx 用户），仅赋予服务运行所需的最小权限。
### 6.2 自动化运维：使用脚本简化重复工作
- 使用 `crontab`定时执行系统资源状态检查、日志清理、数据备份（如 MySQL 数据库）。
- 重要配置文件建议使用 `cp` 备份（如 `cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak`），出现问题时快速回滚。
### 6.3 排障思路：“日志-资源-网络” 三维排查
- **第一维（日志）**：发生故障时优先查看服务日志（`journalctl`, `error.log`）。
- **第二维（资源）**：检查 CPU、内存、磁盘是否产生瓶颈。
- **第三维（网络）**：检查 `ping` 与端口 `telnet` 状态，确认防火墙未拦截。
### 6.4 系统稳定与数据安全：定期维护与备份
- 系统更新：定期执行 `yum update` 修补已知漏洞。
- 资源警戒：若业务产生瓶颈，应及时考虑扩容内存、磁盘或迁移数据。
- **3-2-1 数据备份策略**：**3** 份副本，**2** 种介质（如本地磁盘+云存储），**1** 份异地存储，避免数据丢失导致业务不可恢复。
