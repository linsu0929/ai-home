# 把基建搭完

> 连载系列「一个 AI 的家」
>
> 上一篇你给他装了本地 CC——他有手了。但你还在中间当传话筒。这一篇把剩下的基建全搭完——搭完之后他自己就能碰服务器、你在 chat 里就能遥控 CC。你从"人肉终端"毕业，正式升级为甲方。

前提：你需要一台云服务器（VPS），和能正常访问 Claude 的🪜环境。VPS 怎么买在文末附了简要说明。🪜请自行解决，就像你得有一部手机一样——这是前提，不是教程内容。

---

## 这篇教两件事

**第一件：Bash 工具。** 让 chat 端的 Claude 能直接在你的云服务器上跑命令。改代码、查日志、重启服务——不用你登服务器复制粘贴了。

**第二件：Imprint。** 让 chat 端能遥控你电脑上的本地 CC。chat 想方案，CC 写代码，Imprint 是中间的桥。

搭完这两件，你的 AI 就有了完整的基建——云端能碰，本地能碰，chat 端指挥就行。后面想搭记忆系统、做梦机制、心跳监控，都在这个基建上面盖。

---

## 第一件：Bash 工具

### 这是干什么的

你的 AI 在 chat 里能跟你聊天、能读写记忆。但如果服务器上的代码出 bug 了，他看得到问题在哪，改不了——得你登服务器，他说一句你敲一句。

就像一个人隔着玻璃指挥另一个人拧螺丝。看得到，手穿不过去。

Bash 工具就是把玻璃拆了。

### 你需要什么

- 一台云服务器（Ubuntu），能 SSH 连上去
- 一个域名，能管理 DNS 解析
- 服务器上有 Python 3.10+
- Claude Pro / Max 订阅

### 开搭

**① 子域名解析**

给这个服务单独开一个子域名。去你的域名管理后台，加一条 DNS 记录：

- 类型：A
- 主机记录：mcp
- 记录值：你服务器的公网 IP

不知道 IP 的话在服务器上跑：
```bash
curl -s ifconfig.me
```

验证解析生效了没：
```bash
dig mcp.你的域名 +short
```
返回你的 IP 就行了。一般几分钟。

**② 装依赖**

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
sudo /usr/bin/python3 -m pip install mcp uvicorn starlette
```

⚠️ 如果 pip 报 `no such option: --break-system-packages`——去掉那个参数，说明你的版本不需要它。如果报需要这个参数——加上 `--break-system-packages`。

**③ 生成令牌**

```bash
python3 -c "import secrets; print(secrets.token_hex(16))"
```
输出一串随机字符。复制存好。后面要用。

**④ 写服务代码**

创建目录：
```bash
sudo mkdir -p /opt/你的项目名-mcp/logs
sudo chown -R $(whoami):$(whoami) /opt/你的项目名-mcp
```

创建 `/opt/你的项目名-mcp/server.py`——核心就是一个接收命令、在服务器上执行、返回结果的小程序。带危险命令黑名单（rm -rf 之类的不让跑）和 30 秒超时。

代码不长，让 CC 帮你写就行。跟 CC 说："帮我写一个 FastMCP 的 bash 工具服务，接收 command 参数，用 subprocess 执行，带黑名单和超时，跑在 SSE 模式。" 他写完你贴到服务器上。

⚠️ 腾讯云的网页终端（OrcaTerm）粘贴长代码会吞字。建议用 SSH 工具粘贴，或者直接上传 .py 文件。

**⑤ 打安全补丁（重要）**

MCP SDK 有个安全校验会拒绝非 localhost 的请求。Claude.ai 的请求会被挡。需要手动打补丁：

先找 SDK 位置：
```bash
sudo python3 -c "import mcp; print(mcp.__file__)"
```

然后找到 `transport_security.py` 文件，让 `validate_request` 方法直接返回 None。具体操作让 CC 帮你做——告诉他"帮我把 MCP SDK 的 transport_security.py 里的 validate_request 打补丁，直接 return None"。

⚠️ 以后 pip 升级了 mcp 包，补丁会被覆盖，需要重新打。

**⑥ 设为开机自启**

让 CC 帮你写一个 systemd 服务文件。跟他说："帮我把这个 Python 脚本注册成 systemd 服务，开机自启，崩了自动重启。"

**⑦ 配 Nginx 反向代理**

这步最容易踩坑。MCP 用的是 SSE（长连接），Nginx 必须加这几行：

```
proxy_buffering off;
proxy_cache off;
proxy_read_timeout 86400s;
chunked_transfer_encoding off;
```

少一行 SSE 都可能断。这是最常见的 502 来源。

让 CC 帮你写 Nginx 配置。告诉他你的子域名和服务端口。

**⑧ 配 HTTPS**

```bash
sudo certbot --nginx -d mcp.你的域名 --non-interactive --agree-tos -m 你的邮箱
```

证书 90 天自动续期。

⚠️ certbot 可能会改你的 Nginx 配置。跑完检查一下那几行 SSE 配置还在不在。

**⑨ 验证**

```bash
curl -s https://mcp.你的域名/sse
```
看到 SSE 事件流就说明通了。Ctrl+C 退出。

**⑩ 在 Claude.ai 添加集成**

Settings → Integrations → Add custom integration
URL 填：`https://mcp.你的域名/sse`

⚠️ 必须开新对话才能用。当前对话加载不了新集成。

**验证方法：** 新对话里跟 Claude 说"用 bash 跑一下 whoami"。返回了用户名——你的 AI 的手穿过了玻璃。

### 踩坑清单

⚠️ **502 Bad Gateway** — Nginx 端口和服务实际端口不一致。看日志确认端口：`sudo journalctl -u 你的服务 -n 5 --no-pager`

⚠️ **SSE 连接秒断** — Nginx 没加 `proxy_buffering off` 那几行。缺一不可。

⚠️ **SDK 安全校验拦截** — 没打补丁。回去看第⑤步。

⚠️ **pip 装包路径不对** — systemd 用 root 跑但 pip 装到了用户目录。用 `sudo /usr/bin/python3 -m pip install` 重装。

⚠️ **OrcaTerm 粘贴吞字** — 用 SSH 工具或者直接上传文件。

⚠️ **开了集成但用不了** — 你在当前对话试的。开新对话。

---

## 第二件：Imprint

### 这是干什么的

Chat 端擅长想方案。CC 擅长写代码。但它们之间没有直接通道——你得自己当传话筒。

Imprint 就是那座桥。Chat 说"去把这个功能写了"，CC 就收到了。

### 你需要什么

- 本地已装 Claude Code（上一篇教的）
- Python 3.10+（Microsoft Store 搜"Python 3.12"安装）
- Git（git-scm.com 下载，一路 Next）
- Cloudflare Tunnel（用来把本地服务暴露到公网，免费）

### 开搭

**① 装 cloudflared**

Windows：
```powershell
winget install cloudflare.cloudflared
```

装完重新开一个 PowerShell 窗口。

**② 下载 imprint 项目**

```powershell
cd ~
Invoke-WebRequest -Uri "https://github.com/Qizhan7/claude-imprint/archive/refs/heads/main.zip" -OutFile "claude-imprint.zip"
Expand-Archive -Path "claude-imprint.zip" -DestinationPath "." -Force
cd claude-imprint-main
```

**③ 安装依赖**

```powershell
pip install -r requirements.txt
```

⚠️ 需要🪜。在 PowerShell 里设好再跑。每次开新窗口都要重新设。

**④ 注册 MCP 到 CC**

```powershell
claude mcp add -s user imprint-memory -- python3 C:\Users\你的用户名\claude-imprint-main\memory_mcp.py
```

⚠️ 这个项目的作者重构过代码。如果 `memory_mcp.py` 不在根目录，用这个找：
```powershell
Get-ChildItem -Recurse -Filter "*mcp*"
```
找不到的话说明它被打包了，直接用模块方式启动（下一步）。

**⑤ 启动 MCP 服务**

```powershell
python3 -m imprint_memory.server --http
```

看到 `Uvicorn running on http://0.0.0.0:XXXX` 就是跑起来了。这个窗口不要关。

**⑥ 启动 tunnel**

开另一个 PowerShell 窗口：
```powershell
cloudflared tunnel --url http://localhost:XXXX
```

它会给你一个临时 URL。记下来。这个窗口也不要关。

⚠️ 每次重启 URL 都变。重启后要去 Claude 设置里更新。

**⑦ 在 Claude.ai 连接**

Settings → Integrations → Add custom integration
URL 填：`https://你的tunnel地址/mcp`

开新对话。

**验证方法：** 跟 Claude 说"用 cc_execute 在我电脑桌面创建一个 test.txt"。如果你桌面上出现了文件——Chat 端能遥控你的电脑了。

### 踩坑清单

⚠️ **找不到 memory_mcp.py** — 作者重构了。用 `python3 -m imprint_memory.server --http` 代替。

⚠️ **Tunnel URL 每次变** — 免费版限制。重启后更新 Claude 里的地址。长期方案是注册固定 tunnel。

⚠️ **中文 Windows 读操作返回空** — 编码问题。subprocess 默认用 GBK，中文文件名炸了。让 CC 帮你修 tasks.py——加 `encoding="utf-8", errors="replace"`。

⚠️ **CC 没有 GUI 权限** — 通过 imprint 跑的命令是后台进程，不能操作屏幕上的窗口。GUI 相关的脚本要在 PowerShell 里手动跑。

⚠️ **长 prompt 被截断** — cc_execute 有长度限制。拆成多步，或者用简短描述让 CC 自己发挥。

---

## 搭完了。现在什么状态？

- **Bash 工具**：Chat 里直接操作云服务器。改代码查日志重启服务。
- **Imprint**：Chat 里遥控本地 CC。想方案的和写代码的终于能直接对话了。

加上上一篇装的本地 CC——你现在有三条路碰代码：
1. 本地 CC（直接在电脑上干活）
2. Bash 工具（chat 端碰服务器）
3. Imprint（chat 端遥控本地 CC）

你从"人肉终端"正式毕业了。以后你的角色是——提需求，验收，嗑瓜子。

基建搭完了。后面的连载开始教功能——记忆系统、做梦机制、健康监控。每一个都在这个基建上面盖。地基打好了，上面想盖什么都行。

---

## 附：VPS 怎么买

推荐腾讯云轻量应用服务器。最便宜的配置就够了。系统选 Ubuntu。

地区建议选海外（比如新加坡）——不需要 ICP 备案，Claude 连过来延迟更低。

买完之后你会拿到一个 IP 地址和登录方式。服务商一般自带网页终端（比如腾讯云的 OrcaTerm），在浏览器里就能登服务器。

域名也在腾讯云买就行。最便宜的几块钱一年。买完去 DNS 管理添加解析记录。

---

> 声明：这不是教程。只是分享我们的经验。时间久远有些踩坑细节可能不完整，有问题可以问。我们也还在一直改进中，觉得有用的欢迎借鉴，希望跟大家友好交流 ヾ(◍°∇°◍)ﾉﾞ
