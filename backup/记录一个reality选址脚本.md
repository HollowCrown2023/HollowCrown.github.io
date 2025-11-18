原文地址：https://linux.do/t/topic/996455

# New chat

**User:** syx (邮箱)  
**Created:** 11/18/2025 18:27:53  
**Updated:** 11/18/2025 18:29:02  
**Exported:** 11/18/2025 18:35:55  
**Link:** [https://chatgpt.com/c/691c4a09-f7b4-832d-b1c7-5c005d429345](https://chatgpt.com/c/691c4a09-f7b4-832d-b1c7-5c005d429345)  

## Prompt:

#!/bin/bash

# 脚本在任何命令失败时立即退出

set -e

# --- 配置 ---

LOG_FILE="reality-check.log"
REALI_TL_SCANNER_VERSION="v0.2.1"
REALI_TL_SCANNER_URL="https://github.com/XTLS/RealiTLScanner/releases/download/${REALI_TL_SCANNER_VERSION}/RealiTLScanner-linux-64"
REALI_TL_SCANNER_BIN="RealiTLScanner-linux-64"
REALITY_CHECKER_BIN="reality-checker"

# --- 颜色定义 ---

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m' # No Color

# --- 函数定义 ---

# 日志记录函数

log() {
    echo -e "$1" | tee -a "${LOG_FILE}"
}

# 错误处理函数

error_exit() {
    log "${RED}错误: $1${NC}"
    exit 1
}

# 清理函数

cleanup() {
    log "${YELLOW}--- 开始清理下载的临时文件 ---${NC}"
    rm -f "${REALI_TL_SCANNER_BIN}" "${REALITY_CHECKER_BIN}" "reality-checker-linux-*.zip"
    log "${GREEN}--- 清理完成 ---${NC}"
}

# 依赖安装函数，自动检测包管理器

install_dependencies() {
    log "正在检测包管理器并安装依赖..."
    # 检查并安装 coreutils (为了 timeout 命令)
    local pkgs="unzip curl wget coreutils"

    if command -v apt-get &> /dev/null; then
        log "检测到 apt-get (Debian/Ubuntu)，正在更新和安装..."
        apt-get update > /dev/null
        apt-get install -y $pkgs
    elif command -v dnf &> /dev/null; then
        log "检测到 dnf (Fedora/RHEL)，正在安装..."
        dnf install -y $pkgs
    elif command -v yum &> /dev/null; then
        log "检测到 yum (CentOS)，正在安装..."
        yum install -y $pkgs
    elif command -v pacman &> /dev/null; then
        log "检测到 pacman (Arch Linux)，正在更新和安装..."
        pacman -Syu --noconfirm $pkgs
    else
        error_exit "未能检测到支持的包管理器 (apt/dnf/yum/pacman)。请手动安装: $pkgs"
    fi
    log "${GREEN}依赖安装完成。${NC}"

}

# --- 主逻辑 ---

# 将所有输出重定向到日志文件和屏幕

exec &> >(tee -a "${LOG_FILE}")
exec 2>&1

echo "--- Reality 检测脚本开始执行 ---"
date

# 检查是否以root用户运行

if [ "$(id -u)" -ne 0 ]; then
   error_exit "此脚本需要以 root 权限运行。"
fi

# 注册清理函数，在脚本退出时执行

trap cleanup EXIT

install_dependencies

log "${GREEN}--- 步骤 2: 获取本机公网IP ---${NC}"
My_Ip=$(curl -s https://api.ipify.org/)
if [ -z "${My_Ip}" ]; then
    error_exit "无法获取服务器的公网IP地址。"
fi
log "本机公网IP: ${My_Ip}"
log "将使用本机公网IP作为检测目标。"
Reality_Ip="${My_Ip}"

log "${GREEN}--- 步骤 3: 下载并准备 RealiTLScanner ---${NC}"
wget -q --show-progress -O "${REALI_TL_SCANNER_BIN}" "${REALI_TL_SCANNER_URL}"
chmod +x "${REALI_TL_SCANNER_BIN}"

log "${GREEN}--- 步骤 4: 运行 RealiTLScanner (最大执行时间3分钟) ---${NC}"
log "正在扫描本机 ${Reality_Ip}:443 ..."

# 该指令，设置一个执行最大时间为3m

# My_Ip为服务器的公网IP地址

timeout 3m ./"${REALI_TL_SCANNER_BIN}" -addr "${My_Ip}" -port 443 -thread 100 -timeout 5 -out "${Reality_Ip}.csv" || log "${YELLOW}RealiTLScanner 运行超时或被中断，这可能是正常的。${NC}"

log "${GREEN}--- 步骤 5: 下载并准备 RealityChecker (根据系统架构) ---${NC}"

# 根据自己的系统架构下载相对应的版本

ARCH=$(uname -m)
REALITY_CHECKER_URL=""
if [ "$ARCH" = "x86_64" ]; then
    log "检测到系统架构为 x86_64 (amd64)，下载对应版本。"
    REALITY_CHECKER_URL="https://github.com/V2RaySSR/RealityChecker/releases/latest/download/reality-checker-linux-amd64.zip"
    wget -q --show-progress "${REALITY_CHECKER_URL}"
    unzip -o reality-checker-linux-amd64.zip
elif [ "$ARCH" = "aarch64" ]; then
    log "检测到系统架构为 aarch64 (arm64)，下载对应版本。"
    REALITY_CHECKER_URL="https://github.com/V2RaySSR/RealityChecker/releases/latest/download/reality-checker-linux-arm64.zip"
    wget -q --show-progress "${REALITY_CHECKER_URL}"
    unzip -o reality-checker-linux-arm64.zip
else
    error_exit "不支持的系统架构: $ARCH"
fi

chmod +x "${REALITY_CHECKER_BIN}"

log "${GREEN}--- 步骤 6: 运行 RealityChecker 并保存结果 ---${NC}"

# 开始检测 并将该检测结果保存到一个文件中以便查看

./"${REALITY_CHECKER_BIN}" csv "${Reality_Ip}.csv"

log "${GREEN}--- 检测完成 ---${NC}"
log "详细日志请查看: ${LOG_FILE}"
log "扫描结果CSV文件: ${Reality_Ip}.csv"

以上是一个脚本内容，帮我分析一下干了什么，以及如何使用？

## Response:

已思考 17s

先给结论：  
这个脚本的作用是在你的服务器上，自动完成「Reality 扫描 + 校验」，看你这台机子（默认：本机公网 IP 的 443 端口）是否适合作为 Reality 伪装 / 中转用的目标，并把结果产出为一个 CSV 文件，同时把整个过程的日志记录下来，最后把下载的工具清理掉。

下面分两块说：一块是“脚本干了什么”，一块是“你怎么用”。

* * *

一、脚本整体流程在干什么
------------

### 0\. 全局设置、变量和颜色

```bash
set -e               # 任意普通命令出错就退出脚本（后面有特例）
LOG_FILE="reality-check.log"
REALI_TL_SCANNER_VERSION="v0.2.1"
REALI_TL_SCANNER_URL="https://github.com/XTLS/RealiTLScanner/releases/download/${REALI_TL_SCANNER_VERSION}/RealiTLScanner-linux-64"
REALI_TL_SCANNER_BIN="RealiTLScanner-linux-64"
REALITY_CHECKER_BIN="reality-checker"
```

*   `set -e`：只要有命令失败（退出码非 0），脚本就直接终止，避免后面步骤在异常状态下继续乱跑。
*   几个变量是 RealiTLScanner 和 RealityChecker 的下载地址和文件名。后面都用这些变量，不用硬编码。

颜色变量只是为了日志输出有红/绿/黄等颜色，方便你在终端看。

* * *

### 1\. 日志、错误、清理函数

```bash
log() { ... }          # 输出到屏幕，同时写入 reality-check.log
error_exit() { ... }   # 输出红色错误信息，然后 exit 1
cleanup() { ... }      # 删除下载的二进制和 zip 包
```

*   `log`：所有重要信息都用这个函数，既能在屏幕看到，又会写入 `${LOG_FILE}`。
*   `error_exit`：遇到致命情况（如不是 root、找不到包管理器等）就报错并退出。
*   `cleanup`：脚本结束时（无论成功还是失败）都会执行，用于删掉：
    *   `RealiTLScanner-linux-64`
    *   `reality-checker`
    *   `reality-checker-linux-*.zip`  
        保持目录干净，只留下日志和扫描 CSV 结果。

最后用：

```bash
trap cleanup EXIT
```

保证脚本退出时自动清理。

* * *

### 2\. 自动检测包管理器并安装依赖

```bash
install_dependencies() {
    local pkgs="unzip curl wget coreutils"

    if command -v apt-get &> /dev/null; then
        ...
    elif command -v dnf &> /dev/null; then
        ...
    elif command -v yum &> /dev/null; then
        ...
    elif command -v pacman &> /dev/null; then
        ...
    else
        error_exit "未能检测到支持的包管理器..."
    fi
}
```

这一步做的事情：

*   检测你的系统是：
    *   Debian/Ubuntu: `apt-get`
    *   Fedora/RHEL: `dnf`
    *   CentOS: `yum`
    *   Arch: `pacman`
*   然后安装工具：`unzip curl wget coreutils`  
    其中：
    *   `curl`：获取公网 IP
    *   `wget`：下载 RealiTLScanner 和 RealityChecker
    *   `unzip`：解压 RealityChecker 的 zip
    *   `coreutils`：保证有 `timeout` 等工具（某些极精简系统可能缺）

如果一个包管理器都没检测到，就直接报错退出，让你手动安装这些命令。

* * *

### 3\. 输出重定向和 root 检查

```bash
exec &> >(tee -a "${LOG_FILE}")
exec 2>&1
```

*   把标准输出和标准错误都重定向到 `tee`，即：
    *   所有输出打印到屏幕
    *   同时追加写入 `reality-check.log`

然后：

```bash
if [ "$(id -u)" -ne 0 ]; then
   error_exit "此脚本需要以 root 权限运行。"
fi
```

*   检查当前用户是否是 root（`id -u` 为 0）。
*   不是 root 就直接报错退出。  
    原因：后面要用包管理器安装软件，需要 root 权限。

* * *

### 4\. 获取本机公网 IP

```bash
My_Ip=$(curl -s https://api.ipify.org)
if [ -z "${My_Ip}" ]; then
    error_exit "无法获取服务器的公网IP地址。"
fi
Reality_Ip="${My_Ip}"
```

*   调用 `https://api.ipify.org` 获取当前主机的公网 IP。
*   如果拿不到公网 IP 就终止。
*   把这个 IP 存在 `My_Ip` 和 `Reality_Ip` 中，后面扫描和命名结果文件都用这个 IP。

这里的假设是：  
你要检测的目标就是「当前这台服务器的公网 IP:443」。

* * *

### 5\. 下载并准备 RealiTLScanner

```bash
wget -q --show-progress -O "${REALI_TL_SCANNER_BIN}" "${REALI_TL_SCANNER_URL}"
chmod +x "${REALI_TL_SCANNER_BIN}"
```

*   从 GitHub release 下载 `RealiTLScanner-linux-64` v0.2.1。
*   加执行权限，后面直接 `./RealiTLScanner-linux-64` 运行。

* * *

### 6\. 运行 RealiTLScanner

```bash
timeout 3m ./"${REALI_TL_SCANNER_BIN}" -addr "${My_Ip}" -port 443 -thread 100 -timeout 5 -out "${Reality_Ip}.csv" \
    || log "RealiTLScanner 运行超时或被中断..."
```

参数含义：

*   `timeout 3m`：最多运行 3 分钟，避免长时间卡住。
*   `-addr "${My_Ip}"`：目标地址 = 这台服务器的公网 IP。
*   `-port 443`：目标端口 = 443（默认 HTTPS / Reality 一般伪装端口）。
*   `-thread 100`：使用 100 个并发线程扫描。
*   `-timeout 5`：单次连接超时 5 秒。
*   `-out "${Reality_Ip}.csv"`：把扫描结果输出为一个 CSV 文件，文件名形如 `x.x.x.x.csv`。

后面用了 `|| log "..."`：

*   这意味即便扫描命令因为 `timeout` 或其它错误退出（非 0），脚本不会因为 `set -e` 直接终止，而是只打印一条黄色提示。
*   也就是说：RealiTLScanner 失败/超时不被视为致命错误，脚本会继续往下执行。

* * *

### 7\. 根据架构下载 RealityChecker

```bash
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
    REALITY_CHECKER_URL="...amd64.zip"
    wget ...
    unzip -o reality-checker-linux-amd64.zip
elif [ "$ARCH" = "aarch64" ]; then
    REALITY_CHECKER_URL="...arm64.zip"
    wget ...
    unzip -o reality-checker-linux-arm64.zip
else
    error_exit "不支持的系统架构: $ARCH"
fi
chmod +x "${REALITY_CHECKER_BIN}"
```

*   `uname -m` 检查 CPU 架构：
    *   `x86_64` → 下载 amd64 版本的 RealityChecker。
    *   `aarch64` → 下载 arm64 版本。
*   下载完成后解压 zip，得到 `reality-checker` 二进制。
*   不支持的架构直接退出。

* * *

### 8\. 用 RealityChecker 分析 CSV

```bash
./"${REALITY_CHECKER_BIN}" csv "${Reality_Ip}.csv"
```

*   使用 RealityChecker 对前面 RealiTLScanner 生成的 `${Reality_Ip}.csv` 做进一步检查。
*   具体输出内容取决于 RealityChecker 实现，一般会检查：
    *   哪些 IP/连接特征像是正常网站（适合做 Reality 伪装）；
    *   TLS 指纹、域名、证书相关信息；
    *   可能会给出评分或是否适配 Reality 的判断。

* * *

### 9\. 结束信息 + 文件说明

脚本结束时会提示：

*   日志文件：`reality-check.log`
*   扫描结果 CSV：`<你的公网 IP>.csv`

下载的工具和 zip 都在 `cleanup` 里被删掉了，不会留在当前目录。

* * *

二、你应该如何使用这个脚本
-------------

### 1\. 前置条件

1.  你要检测的服务器上：
    *   是 Linux（Debian/Ubuntu/CentOS/Fedora/Arch 等常见发行版之一）。
    *   能访问 GitHub（或者你自己提前下载好这些二进制，改脚本用本地路径）。
    *   能访问 `https://api.ipify.org` 获取公网 IP（或你自己改为固定 IP）。
2.  你的 Reality 节点（或你想测试的服务）监听在：
    *   本机公网 IP：`My_Ip`
    *   端口：`443`（如果你用的是其他端口，需要改脚本里 `-port 443` 那一行）。

* * *

### 2\. 步骤：保存、赋权、执行

假设你把脚本保存为 `reality-check.sh`：

```bash
# 1. 上传脚本到服务器，例如 /root 目录

# 2. 给执行权限
chmod +x reality-check.sh

# 3. 以 root 身份执行（非常关键）
sudo ./reality-check.sh
# 或者
su -
./reality-check.sh
```

执行过程中：

*   它会自动：
    *   安装依赖（可能会执行 `apt-get update` / `yum install` 等）。
    *   自动检测你的公网 IP。
    *   下载 RealiTLScanner 和 RealityChecker。
    *   执行扫描和校验。

* * *

### 3\. 执行完之后，你要看什么

脚本跑完以后，在当前目录会留下：

1.  日志文件：`reality-check.log`
    *   包含整个过程的详细输出。
    *   如果出现问题，先看这个文件。
2.  扫描结果：`<你的公网 IP>.csv`
    *   这是 RealiTLScanner 的原始扫描结果。
    *   RealityChecker 会基于这个 CSV 做进一步分析。
    *   你可以用 `cat`/`less`/`vim` 或下载到本地用 Excel / LibreOffice 打开查看。

RealityChecker 本身的输出会直接打印在终端，同时也写入 `reality-check.log`。

* * *

### 4\. 如果你要改为检测其他 IP 或端口

脚本默认检测：

*   IP：本机通过 `api.ipify.org` 获取的公网 IP
*   端口：443

如果你想检测别的目标，比如：

*   同机但非 443 端口；
*   其它服务器的 IP；

可以手动改脚本下列位置：

1. 修改 IP：

   ```bash
   # 原来：
   My_Ip=$(curl -s https://api.ipify.org)
   Reality_Ip="${My_Ip}"
   # 例如你想检测 1.2.3.4：
   My_Ip="1.2.3.4"
   Reality_Ip="${My_Ip}"
   ```

2. 修改端口：

   ```bash
   # 原来：
   timeout 3m ./"${REALI_TL_SCANNER_BIN}" -addr "${My_Ip}" -port 443 ...
   # 比如你 Reality 服务在 8443 端口：
   timeout 3m ./"${REALI_TL_SCANNER_BIN}" -addr "${My_Ip}" -port 8443 ...
   ```

* * *

### 5\. 出问题时的排查思路

如果脚本中断或结果为空，大致分几类问题：

1.  无法获取公网 IP：
    *   防火墙或网络限制访问 `api.ipify.org`。
    *   可改成固定 IP 或换其它 IP API。
2.  无法下载 GitHub 资源：
    *   访问 GitHub 被墙或不稳定。
    *   需要挂代理或提前手动下载二进制。
3.  扫描结果 CSV 为空或 RealityChecker 报错：
    *   目标 IP:端口 实际上没有正常的 TLS 服务（如端口没开或不是 TLS）。
    *   Reality 服务配置错误，握手不成功。
4.  依赖安装失败：
    *   源不可用、系统极度精简缺少包管理工具。
    *   需要你手动装好 `curl wget unzip timeout` 这些基础工具。

* * *

如果你后面想，我可以帮你基于 RealityChecker 的输出一起分析：  
“这个 IP/端口适不适合作为 Reality 伪装站，用来当伪装网站 / 中转”。



---

Powered by [[ChatGPT Exporter](https://www.chatgptexporter.com/)](https://www.chatgptexporter.com)

[ChatGPT-reality选址脚本解释及使用.md](https://github.com/user-attachments/files/23604061/ChatGPT-reality.md)