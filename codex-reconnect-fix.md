# Mac Clash + Remote Ubuntu 下解决 Codex 回答时反复 reconnect 的问题

## 一、问题现象

我在 Mac 本机使用 Clash 作为代理工具，代理端口为：

```bash
127.0.0.1:7890
```

同时，我通过 SSH 连接远程 Ubuntu 主机 `ubuntu22`，并在远程 Ubuntu 中使用 Codex CLI。

远端 Codex 已经安装完成，版本为：

```bash
codex-cli 0.139.0
```

一开始 Codex 并不是完全不能启动，也不是完全不能回答，而是经常在回答过程中出现类似：

```text
reconnect
reconnecting
connection lost
```

这类反复重连的问题。

表现上就是：

* Codex 可以进入；
* 有时也能开始生成回答；
* 但回答过程中容易中断；
* 经常出现 reconnect；
* 长回答、复杂任务、需要联网的任务更容易出问题。

## 二、问题分析

这个问题的核心不是 Codex 没安装好，而是 **远程 Ubuntu 上的 Codex 没有稳定使用 Mac 本机的 Clash 代理**。

我的 Clash 是运行在 Mac 本机上的，监听地址是：

```bash
127.0.0.1:7890
```

但是 Codex 实际运行在远程 Ubuntu 上。

这就导致一个很容易忽略的问题：

```text
Mac 上的 127.0.0.1 是 Mac 自己；
Ubuntu 上的 127.0.0.1 是远程 Ubuntu 自己。
```

所以，如果在远程 Ubuntu 里直接写：

```bash
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
```

并不能访问到 Mac 上的 Clash。

对远程 Ubuntu 来说，`127.0.0.1:7890` 指的是远程 Ubuntu 自己的 7890 端口，而不是 Mac 的 7890 端口。

因此，虽然 Mac 本机 Clash 是正常的，但远程 Ubuntu 上的 Codex 代理链路并不稳定，最终就表现为 Codex 回答时反复 reconnect。

## 三、解决思路

解决思路是：

> 通过 SSH RemoteForward，在远程 Ubuntu 上开一个本地端口 `7897`，把它转发到 Mac 本机的 Clash 端口 `7890`。

最终链路如下：

```text
远程 Ubuntu Codex
        ↓
127.0.0.1:7897
        ↓ SSH RemoteForward
Mac 127.0.0.1:7890
        ↓
Clash
        ↓
OpenAI / Codex 服务
```

也就是说：

* Mac 本机继续使用 Clash 的 `7890`；
* 远程 Ubuntu 不直接访问 `7890`；
* 远程 Ubuntu 访问自己的 `127.0.0.1:7897`；
* `7897` 通过 SSH 转发到 Mac 的 `7890`。

## 四、环境信息

| 项目          | 配置               |
| ----------- | ---------------- |
| 本机系统        | macOS            |
| 本机代理工具      | Clash            |
| 本机代理端口      | `127.0.0.1:7890` |
| 远程系统        | Ubuntu           |
| SSH 主机别名    | `ubuntu22`       |
| 远端用户        | `tamarisk2`      |
| 远端 Codex 版本 | `0.139.0`        |
| 远端代理入口      | `127.0.0.1:7897` |
| SSH 转发关系    | `7897 → 7890`    |

## 五、具体配置过程

### 1. 确认 Mac 本机 Clash 端口正常

在 Mac 终端执行：

```bash
nc -vz 127.0.0.1 7890
```

如果输出类似：

```text
Connection to 127.0.0.1 port 7890 [tcp/*] succeeded!
```

说明 Mac 本机 Clash 的 `7890` 端口是正常的。

### 2. 配置 SSH RemoteForward

在 Mac 本机编辑 SSH 配置文件：

```bash
nano ~/.ssh/config
```

找到 `ubuntu22` 对应的主机配置段，保留或添加：

```sshconfig
Host ubuntu22
    HostName <your-ubuntu-host>
    User tamarisk2
    IdentityFile ~/.ssh/id_ed25519
    RemoteForward 127.0.0.1:7897 127.0.0.1:7890
    ExitOnForwardFailure yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

其中最关键的是这一行：

```sshconfig
RemoteForward 127.0.0.1:7897 127.0.0.1:7890
```

它的含义是：

```text
在远程 Ubuntu 上监听 127.0.0.1:7897，
并把这个端口转发到 Mac 本机的 127.0.0.1:7890。
```

同时，我把：

```sshconfig
ExitOnForwardFailure no
```

改成了：

```sshconfig
ExitOnForwardFailure yes
```

这样做的好处是：如果端口转发失败，SSH 会直接报错，而不是表面上连接成功，实际代理不可用。

### 3. 重新连接远程 Ubuntu

修改 `~/.ssh/config` 后，已有 SSH 会话不会自动生效。

所以需要退出所有已有的 `ubuntu22` 终端连接和 VS Code Remote SSH 连接，然后重新连接：

```bash
ssh ubuntu22
```

如果使用 VS Code Remote SSH，也需要重新连接远程主机。

### 4. 验证远程 7897 是否可用

进入远程 Ubuntu 后执行：

```bash
nc -vz 127.0.0.1 7897
```

如果输出类似：

```text
Connection to 127.0.0.1 port 7897 [tcp/*] succeeded!
```

说明远程 Ubuntu 上的 `7897` 已经成功建立。

继续测试 HTTP 代理：

```bash
curl -I -x http://127.0.0.1:7897 https://api.openai.com
```

再测试 SOCKS5 代理：

```bash
curl -I -x socks5h://127.0.0.1:7897 https://api.openai.com
```

如果不是 `Connection refused`、`Failed to connect`、`timeout` 这类错误，就说明代理链路基本是通的。

有时返回 `401`、`403`、`404` 并不一定是代理失败，因为这说明请求已经到达目标服务器，只是目标服务本身拒绝了未认证或不完整的请求。

## 六、配置远程 Codex 的代理环境变量

远端原本没有：

```bash
~/.codex/.env
```

所以在远程 Ubuntu 中新建该文件：

```bash
mkdir -p ~/.codex
```

写入代理变量：

```bash
cat > ~/.codex/.env <<'EOF'
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
ALL_PROXY=socks5h://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1,::1

http_proxy=http://127.0.0.1:7897
https_proxy=http://127.0.0.1:7897
all_proxy=socks5h://127.0.0.1:7897
no_proxy=localhost,127.0.0.1,::1
EOF
```

这里同时写入大写和小写两套代理变量，是为了兼容不同程序对代理环境变量的读取习惯。

其中：

```bash
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
```

用于 HTTP / HTTPS 请求。

```bash
ALL_PROXY=socks5h://127.0.0.1:7897
```

用于 SOCKS5 代理。

这里使用 `socks5h`，是为了让 DNS 解析也通过代理进行，避免某些情况下 DNS 解析异常导致连接不稳定。

```bash
NO_PROXY=localhost,127.0.0.1,::1
```

表示访问本地地址时不走代理，避免影响本地服务。

### 7. 设置文件权限

设置 `.env` 文件权限：

```bash
chmod 600 ~/.codex/.env
```

检查权限：

```bash
ls -l ~/.codex/.env
```

预期类似：

```text
-rw------- 1 tamarisk2 tamarisk2 ... /home/tamarisk2/.codex/.env
```

### 8. 检查配置内容

```bash
cat ~/.codex/.env
```

内容应为：

```bash
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
ALL_PROXY=socks5h://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1,::1

http_proxy=http://127.0.0.1:7897
https_proxy=http://127.0.0.1:7897
all_proxy=socks5h://127.0.0.1:7897
no_proxy=localhost,127.0.0.1,::1
```

## 七、验证 Codex

检查 Codex 版本：

```bash
codex --version
```

输出：

```text
codex-cli 0.139.0
```

然后重新启动 Codex：

```bash
codex
```

经过上述配置后，Codex 在远程 Ubuntu 中回答时反复出现 reconnect 的问题得到解决。

## 八、这次真正解决了什么

这次解决的核心问题是：

> 远程 Ubuntu 上的 Codex 在回答过程中反复 reconnect。

不是简单地解决“Codex 无法启动”，也不是单纯解决“Clash 端口不通”，而是把远程 Codex 的网络链路稳定到了：

```text
Codex → Ubuntu 7897 → SSH 转发 → Mac 7890 → Clash
```

这样 Codex 的请求不再走不稳定或错误的网络路径，而是稳定通过 Mac 本机 Clash 代理访问外部服务。

最终效果：

1. Codex 可以正常启动；
2. Codex 回答过程中不再频繁 reconnect；
3. 长回答和联网任务稳定性明显提升；
4. 不需要修改远程 Ubuntu 的系统全局代理；
5. 不影响另一个 SSH 主机 `mylinux`；
6. 配置集中在 `ubuntu22` 和远端用户的 `~/.codex/.env` 中，方便回滚和维护。

## 九、容易踩的坑

### 1. 远程 Ubuntu 不能直接写 7890

错误写法：

```bash
HTTP_PROXY=http://127.0.0.1:7890
```

这个地址只适合 Codex 运行在 Mac 本机时使用。

如果 Codex 运行在远程 Ubuntu，就应该写：

```bash
HTTP_PROXY=http://127.0.0.1:7897
```

因为 `7897` 才是远程 Ubuntu 上通过 SSH 转发出来的代理入口。

### 2. 修改 SSH 配置后必须重新连接

修改 `~/.ssh/config` 后，旧的 SSH 会话不会自动应用新配置。

必须退出旧连接，再重新连接：

```bash
ssh ubuntu22
```

VS Code Remote SSH 也需要重新连接。

### 3. Mac 上 Clash 必须保持开启

远程 Ubuntu 的 `7897` 最终转发到 Mac 的 `7890`。

如果 Mac 上 Clash 没开，远程代理入口虽然可能存在，但最终仍然无法正常访问外部网络。

### 4. `ExitOnForwardFailure yes` 很重要

如果端口转发失败，而 `ExitOnForwardFailure` 仍然是 `no`，SSH 可能会继续连接成功，导致表面上看不出问题。

设置为：

```sshconfig
ExitOnForwardFailure yes
```

可以让转发失败的问题直接暴露出来，排查起来更清楚。

### 5. 不建议写入全局代理

这次只修改了：

```bash
~/.codex/.env
```

没有修改：

```bash
~/.bashrc
~/.zshrc
/etc/environment
```

这样可以避免代理影响其他命令，比如：

```bash
ssh
git
npm
pip
curl
```

如果把代理写到全局环境变量里，可能会影响局域网访问、实验箱连接、其他 SSH 连接等。

## 十、回滚方法

如果不想继续使用该配置，可以删除远程 Ubuntu 上的 Codex 代理文件：

```bash
rm ~/.codex/.env
```

如果想取消 SSH 转发，可以删除或注释 Mac `~/.ssh/config` 中的这一行：

```sshconfig
RemoteForward 127.0.0.1:7897 127.0.0.1:7890
```

然后重新连接 SSH。

## 十一、总结

这次问题的本质是：

```text
Clash 在 Mac 上，Codex 在远程 Ubuntu 上。
```

Mac 的 `127.0.0.1:7890` 不能直接被远程 Ubuntu 使用。

正确方式是通过 SSH RemoteForward 把远程 Ubuntu 的 `127.0.0.1:7897` 转发到 Mac 的 `127.0.0.1:7890`，然后让远程 Codex 使用 `7897` 作为代理入口。

最终记忆方式：

```text
Mac 本机 Clash：7890
远程 Ubuntu Codex：7897
SSH 转发关系：7897 → 7890
Codex 代理配置：~/.codex/.env
```

通过这种方式，我解决了远程 Ubuntu 上 Codex 回答过程中反复 reconnect 的问题，并且没有修改系统全局代理，配置更加清晰，也更容易维护。
