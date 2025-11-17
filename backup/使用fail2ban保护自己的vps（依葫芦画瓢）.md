这是原文的链接：https://www.nodeseek.com/post-499878-1

# 粘贴复制原文：
教程：如何使用 fail2ban 保护你的 Linux 服务器
如果你在服务器的 lastb 日志中看到了成千上万条失败的登录记录，那么你正在遭受 SSH 暴力破解攻击。fail2ban 是一个自动化的“安保系统”，它会监控这些日志，并自动封禁那些恶意尝试的 IP 地址。

本教程将指导你如何安装、配置 fail2ban，并设置一个严格且安全的策略。

什么是 fail2ban？
你可以把 fail2ban 想象成一个“数字保镖”。

它会实时读取你服务器的日志（如 /var/log/auth.log）。
当你设置的“规则”（Jail）被触发时（例如：10 分钟内，同一个 IP 登录 SSH 失败 3 次）。
它会立即调用防火墙（iptables）自动添加一条规则，将该 IP 封禁一段时间（例如 1 天）。
步骤 1：安装 fail2ban
首先，更新你的包列表并安装 fail2ban：

sudo apt update
sudo apt install fail2ban -y
安装完成后，fail2ban 会自动启动并运行。我们还要设置它开机自启：

sudo systemctl start fail2ban
sudo systemctl enable fail2ban
此时，fail2ban 已经在用它的默认配置保护你的 SSH 了。但默认配置通常太宽松，我们需要自定义它。

步骤 2：创建你自己的配置文件（jail.local）
这是最重要的一步。fail2ban 的默认配置在 /etc/fail2ban/jail.conf。

⚠️ 警告：永远不要直接修改 jail.conf 文件！
因为 fail2ban 将来升级时，会覆盖这个文件，你所有的修改都会丢失。

正确的做法是创建一个 jail.local 文件，这个文件中的配置会覆盖 jail.conf 中的同名设置，并且永远不会被升级覆盖。

# 使用 nano（或 vim）创建并编辑这个新文件
sudo nano /etc/fail2ban/jail.local
步骤 3：配置你的安全策略
将下面的全部内容复制并粘贴到你刚刚打开的 jail.local 文件中。每一项的含义都已在注释中详细说明。

[DEFAULT]
#
# (重要) 白名单设置 (ignoreip):
# ----------------------------------------------------
# 用空格分隔的IP地址或网段列表。这些地址永远不会被封禁。
# 127.0.0.1/8 ::1 是本地回环地址，必须保留。
#
# !! 把 "YOUR_STATIC_IP_HERE" 替换成你自己的公网IP !!
# (在本地浏览器搜 "what is my ip" 找到它)
#
ignoreip = 127.0.0.1/8 ::1 YOUR_STATIC_IP_HERE

#
# 封禁时间 (bantime):
# ----------------------------------------------------
# 默认封禁 10 分钟太短了，攻击者很快就会回来。
# 1d = 1 天
# 1h = 1 小时
#
bantime = 1d

#
# 寻找时间 (findtime):
# ----------------------------------------------------
# 在这段时间内达到 "maxretry" 次数就会被封禁。
#
findtime = 10m

#
# 最大尝试次数 (maxretry):
# ----------------------------------------------------
# 默认的 5 次太宽容了。我们设为 3 次。
#
maxretry = 3

#
# [sshd] - 针对 SSH 服务的特定设置
# ----------------------------------------------------
# 默认情况下 [DEFAULT] 区的设置会应用到所有服务，
# 我们在这里明确启用 sshd 监狱（Jail）。
#
[sshd]
enabled = true
配置说明：
我们这份配置的含义是：

“对于 SSH 服务，如果任何一个 IP 地址（不在白名单 ignoreip 内）在 10 分钟（findtime）内登录失败达到 3 次（maxretry），则立即封禁该 IP 1 天（bantime）。”

保存并退出：
在 nano 编辑器中，按 Ctrl + O，回车（Enter）保存，然后按 Ctrl + X 退出。

步骤 4：重启 fail2ban 使配置生效
这是必须的一步，否则你的 jail.local 配置不会被加载。

sudo systemctl restart fail2ban
步骤 5：验证 fail2ban 是否在工作
等待几分钟，让 fail2ban 开始读取日志并抓捕那些攻击者。然后运行以下命令：

# 检查 sshd "监狱" (Jail) 的状态
sudo fail2ban-client status sshd
✅ 输出示例（刚启动时）：
Status for the jail: sshd
|- Filter
| |- Currently failed: 0
| |- Total failed: 0
| `- File list: /var/log/auth.log
`- Actions
|- Currently banned: 0
|- Total banned: 0
`- Banned IP list:
✅ 运行一段时间后（成功封禁）：
Status for the jail: sshd
|- Filter
| |- Currently failed: 2
| |- Total failed: 1392
| `- File list: /var/log/auth.log
`- Actions
|- Currently banned: 14 <-- 成功封禁了 14 个 IP
|- Total banned: 58
`- Banned IP list: 116.88.x.x 104.22.x.x ...
当你看到 Banned IP list 中出现 IP 地址时，恭喜你，你的“安保系统”已经开始自动工作了！

（附）紧急情况：如何解封你自己的 IP？
如果你不小心把自己锁在了外面（例如，你没有设置白名单，然后在手机上输错了 3 次密码），你需要通过其他方式（如云服务商的 VNC 控制台）登录服务器，然后执行以下命令：

# 假如你不小心封了 IP "8.8.8.8"
sudo fail2ban-client set sshd unbanip 8.8.8.8
这条命令会立即将该 IP 从 sshd 监狱中解封。

# 补充：
我的几个vps都是装的Ubuntu，按照原帖的内容无法正确重启，以下是我修改了的内容：
[DEFAULT]
#
#(重要)白名单设置(ignoreip):
# ----------------------------------------------------
# 用空格分隔的IP地址或网段列表。这些地址永远不会被封禁。
# 127.0.0.1/8 ::1 是本地回环地址，必须保留
#
## !! 把 "YOUR_STATIC_IP_HERE" 替换成你自己的公网IP !!
# (在本地浏览器搜 "what is my ip" 找到它)
#
ignoreip = 127.0.0.1/8 ::1 144.34.237.48 45.12.90.139 107.172.238.201 154.17.3.170
#
# 封禁时间 (bantime):
# ----------------------------------------------------
# 默认封禁 10 分钟太短了，攻击者很快就会回来。
# 1d = 1 天
# 1h = 1 小时
#
bantime = 1d
## 寻找时间 (findtime):
# ----------------------------------------------------
# 在这段时间内达到 "maxretry" 次数就会被封禁。
#
findtime = 10m
# 最大尝试次数 (maxretry):
# ----------------------------------------------------
# 默认的 5 次太宽容了。我们设为 3 次。
maxretry = 5
# [sshd] - 针对 SSH 服务的特定设置
# ----------------------------------------------------
# 默认情况下 [DEFAULT] 区的设置会应用到所有服务，
# 我们在这里明确启用 sshd 监狱（Jail）。
[sshd]
enabled = true
backend = systemd

主要是最后一句 backend = systemd ，GPT给的解答如下：
报错已经变成：

Have not found any log file for sshd jail

说明当前问题是：[sshd] 这个 jail 被启用了，但 Fail2ban 找不到它要监控的 SSH 日志文件，所以启动直接失败退出。

本质上有两种典型情况：

Fail2ban 按“看文件”的方式工作，但你系统上没有 /var/log/auth.log（或对应的 sshd 日志文件）；

后端本来应该用 systemd 读 journal，却没有配置好/没启用。

你现在的 jail.local 只有：

[sshd]
enabled = true

启用了 jail，但没有告诉它“从哪里看日志”，于是它去继承默认配置。如果默认配置指向了一个不存在的日志文件，就会报这个错。
下面给一个稳妥的修改方案。

第一步：修改 [sshd]，强制使用 systemd 后端

在多数现代 Debian/Ubuntu 系统上，SSH 日志写在 systemd 的 journal 里，用 Fail2ban 内置的 systemd 后端最简单。

编辑配置：

sudo nano /etc/fail2ban/jail.local


把末尾改成类似下面这样（连同前面的 DEFAULT 一并给你一个完整参考）：

[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 144.34.237.48 45.12.90.139 107.172.238.201 154.17.3.170
bantime  = 1d
findtime = 10m
maxretry = 5

[sshd]
enabled = true
backend = systemd
# 不在这里写 logpath，让它走 systemd 后端读 journal


保存退出后，重启 Fail2ban：

sudo systemctl restart fail2ban
sudo systemctl status fail2ban

如果一切正常，应看到：

Active: active (running)