# Mac + Ubuntu Remote SSH 下配置 Codex VS Code 插件与桌面端远程项目代理连接

本文记录一次在 Mac 本机使用代理，通过 SSH 连接远程 Ubuntu，并让 VS Code 中的 Codex 插件以及 Codex 桌面端远程项目都能正常联网的完整过程。

## 1. 背景

我的开发环境大致如下：

- 本机：Mac
- 远程机器：Ubuntu 20.04 / Ubuntu 22.04
- 远程连接方式：SSH / VS Code Remote SSH
- 本机代理：Clash，监听 `127.0.0.1:7890`
- 远程代理入口：Ubuntu 上的 `127.0.0.1:7897`
- 目标：
  - VS Code Remote 中的 Codex 插件可以联网
  - Codex 桌面端可以打开远程 Ubuntu 项目
  - Linux 终端中的 `codex` 命令可以正常联网

核心思路是：

```text
Mac 本机代理 127.0.0.1:7890
        ↓
SSH RemoteForward
        ↓
Ubuntu 远程代理入口 127.0.0.1:7897
        ↓
VS Code / Codex / curl / git / npm 走代理联网
```

## 2. 配置 Mac SSH RemoteForward

在 Mac 上编辑 SSH 配置：

```bash
nano ~/.ssh/config
```

加入远程主机配置：

```sshconfig
Host ubuntu22
    HostName <你的远程Ubuntu地址>
    User <你的Ubuntu用户名>
    IdentityFile ~/.ssh/id_ed25519

    RemoteForward 127.0.0.1:7897 127.0.0.1:7890
    ExitOnForwardFailure no

    ServerAliveInterval 60
    ServerAliveCountMax 3
```

其中最关键的是：

```sshconfig
RemoteForward 127.0.0.1:7897 127.0.0.1:7890
```

含义是：

```text
远程 Ubuntu 的 127.0.0.1:7897
转发到
Mac 本机的 127.0.0.1:7890
```

也就是说，远程 Ubuntu 访问：

```text
http://127.0.0.1:7897
```

实际上是在走 Mac 上的 Clash 代理。

测试 SSH 配置：

```bash
ssh -G ubuntu22 | egrep -i 'hostname|user|identityfile|remoteforward|exitonforwardfailure'
```

测试连接：

```bash
ssh ubuntu22 'echo SSH_OK'
```

看到：

```text
SSH_OK
```

说明 SSH 连接正常。

## 3. 测试远程代理是否生效

先确认 Mac 本机代理端口正常：

```bash
nc -vz 127.0.0.1 7890
```

然后测试 Ubuntu 上的远程代理入口：

```bash
ssh ubuntu22 'curl -I -x http://127.0.0.1:7897 https://github.com | head'
```

如果看到：

```text
HTTP/1.1 200 Connection established
HTTP/2 200
```

说明远程 Ubuntu 已经可以通过 `127.0.0.1:7897` 走代理访问 GitHub。

测试 ChatGPT：

```bash
ssh ubuntu22 'curl -I -x http://127.0.0.1:7897 https://chatgpt.com | head'
```

可能会看到：

```text
HTTP/1.1 200 Connection established
HTTP/2 403
```

这里的 `403` 不一定代表代理失败，通常是 Cloudflare 对命令行请求的拦截。只要前面有：

```text
HTTP/1.1 200 Connection established
```

就说明代理链路已经建立。

## 4. 配置 VS Code Remote 中的 Codex 插件

VS Code 的 Codex 插件主要读取 VS Code Remote 的代理设置。

连接远程 Ubuntu 后，在 VS Code 中打开：

```text
Cmd + Shift + P
```

搜索：

```text
Preferences: Open Remote Settings (JSON)
```

注意一定要打开 Remote Settings，而不是本机 User Settings。

加入：

```json
{
    "http.proxy": "http://127.0.0.1:7897",
    "http.proxyStrictSSL": false
}
```

如果原来已有配置，需要合并，例如：

```json
{
    "editor.fontSize": 14,
    "http.proxy": "http://127.0.0.1:7897",
    "http.proxyStrictSSL": false
}
```

然后重启 VS Code Remote：

```text
Cmd + Shift + P
```

搜索：

```text
Remote-SSH: Kill VS Code Server on Host
```

选择对应远程主机，然后重新连接。

此时 VS Code 中的 Codex 插件应该可以正常联网。

## 5. 为什么 Codex 桌面端还不通？

VS Code 插件通了，不代表 Codex 桌面端远程项目也通。

原因是：

```text
VS Code 插件读取 VS Code Remote 的 settings.json
Codex 桌面端远程项目通过 SSH 调用远程 Linux 上的 codex 命令
```

所以桌面端不一定会读取 VS Code 的：

```json
"http.proxy": "http://127.0.0.1:7897"
```

它更依赖远程 Linux 上 `codex` 命令本身的运行环境。

因此还需要给远程 Linux 上的 `codex` 命令配置代理。

## 6. 安装 Codex CLI

进入远程 Ubuntu：

```bash
ssh ubuntu22
```

临时设置代理：

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897

export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897
export all_proxy=http://127.0.0.1:7897
```

安装 Codex CLI：

```bash
npm install -g @openai/codex
```

安装完成后测试：

```bash
command -v codex
codex --version
```

正常会看到类似：

```text
/usr/local/bin/codex
codex-cli 0.xxx.x
```

## 7. 写入 Linux 环境变量

为了让普通终端也默认走代理，可以把代理变量写进 `~/.profile` 和 `~/.bashrc`。

编辑：

```bash
nano ~/.profile
```

末尾加入：

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897

export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897
export all_proxy=http://127.0.0.1:7897

export NO_PROXY=localhost,127.0.0.1,::1
export no_proxy=localhost,127.0.0.1,::1
```

生效：

```bash
source ~/.profile
```

再编辑：

```bash
nano ~/.bashrc
```

同样加入上面的代理变量：

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897

export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897
export all_proxy=http://127.0.0.1:7897

export NO_PROXY=localhost,127.0.0.1,::1
export no_proxy=localhost,127.0.0.1,::1
```

生效：

```bash
source ~/.bashrc
```

测试：

```bash
env | grep -i proxy
```

## 8. 包装 codex 命令，让桌面端也走代理

有些远程启动方式不一定读取 `~/.profile` 或 `~/.bashrc`。

为了让 Codex 桌面端远程项目稳定走代理，可以把 Linux 上的 `codex` 命令包装一层。

先查看路径：

```bash
command -v codex
ls -l $(command -v codex)
```

如果路径是：

```text
/usr/local/bin/codex
```

可以执行下面脚本：

```bash
CODEX_PATH="$(command -v codex)"

if [ -z "$CODEX_PATH" ]; then
    echo "没有找到 codex，请先安装 Codex CLI"
    exit 1
fi

echo "当前 codex 路径: $CODEX_PATH"

REAL_CODEX="${CODEX_PATH}.real"

if [ ! -e "$REAL_CODEX" ]; then
    if [ -w "$(dirname "$CODEX_PATH")" ]; then
        mv "$CODEX_PATH" "$REAL_CODEX"
    else
        sudo mv "$CODEX_PATH" "$REAL_CODEX"
    fi
fi

if [ -w "$(dirname "$CODEX_PATH")" ]; then
    TEE_CMD="tee"
    CHMOD_CMD="chmod"
else
    TEE_CMD="sudo tee"
    CHMOD_CMD="sudo chmod"
fi

$TEE_CMD "$CODEX_PATH" > /dev/null <<EOF
#!/usr/bin/env bash

export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897

export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897
export all_proxy=http://127.0.0.1:7897

export NO_PROXY=localhost,127.0.0.1,::1
export no_proxy=localhost,127.0.0.1,::1

exec "$REAL_CODEX" "\$@"
EOF

$CHMOD_CMD +x "$CODEX_PATH"

codex --version
```

如果最后还能正常输出版本号，说明包装成功。

这一步完成后，无论手动运行：

```bash
codex
```

还是 Codex 桌面端远程调用 `codex app-server`，都会自动带上代理变量。

## 9. 处理 7897 端口冲突 warning

有时候执行 SSH 命令会出现：

```text
Warning: remote port forwarding failed for listen port 7897
```

这个通常不是 Codex 的错误，而是因为远程 Ubuntu 的 `7897` 已经被另一个 SSH 连接占用了，例如 VS Code Remote 已经建立了这个转发。

可以用下面命令测试，不再重复创建端口转发：

```bash
ssh -o ClearAllForwardings=yes ubuntu22 'ss -ltnp | grep 7897 || echo "7897_NOT_LISTENING"'
```

测试代理：

```bash
ssh -o ClearAllForwardings=yes ubuntu22 'curl -I -x http://127.0.0.1:7897 https://github.com | head'
```

如果能看到：

```text
HTTP/1.1 200 Connection established
HTTP/2 200
```

就说明代理是正常的。

## 10. 重启 Codex 远程进程

如果修改后桌面端仍然不通，可以在远程 Ubuntu 上执行：

```bash
pkill -f codex || true
```

然后完全退出 Codex 桌面端，重新打开，再重新连接远程项目。

## 11. 新 Ubuntu 22.04 复现流程

最短流程如下：

```text
1. Mac 上确认 Clash 代理端口是 7890
2. Mac 的 ~/.ssh/config 中添加 RemoteForward 7897 → 7890
3. ssh ubuntu22 测试连接
4. curl -x http://127.0.0.1:7897 https://github.com 测试代理
5. Ubuntu 上安装 Codex CLI
6. Ubuntu 的 ~/.profile 和 ~/.bashrc 写入代理变量
7. 包装 Ubuntu 上的 codex 命令
8. VS Code Remote Settings JSON 配置 http.proxy
9. Codex 桌面端添加 SSH 远程项目
10. 若异常，使用 ClearAllForwardings 测试端口和代理
```

## 12. 总结

这个问题的关键不只是“代理是否通”，而是要区分不同组件读取代理配置的位置：

```text
curl / git / npm:
    读取环境变量 HTTP_PROXY / HTTPS_PROXY

VS Code Codex 插件:
    读取 VS Code Remote settings.json 中的 http.proxy

Codex 桌面端远程项目:
    通过 SSH 调用远程 Linux 上的 codex 命令
    因此需要保证 codex 命令本身带代理环境变量
```

最终方案是：

```text
Mac Clash 7890
    ↓ SSH RemoteForward
Ubuntu 127.0.0.1:7897
    ↓
VS Code Remote http.proxy
    ↓
Codex 插件通

Ubuntu codex 包装脚本
    ↓
Codex 桌面端远程项目通
```
---

## 相关文章

- [Mac Clash + Remote Ubuntu 下解决 Codex 回答时反复 reconnect 的问题](./codex-reconnect-fix.md)
