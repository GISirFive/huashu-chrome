# 在 WSL 中控制 Windows Chrome —— 部署手册

适用场景：AI agent 跑在 WSL，被控的 Chrome 跑在 Windows 宿主机。
本目录的代码相对上游 `huashu-chrome` 做了改动以支持这个拓扑。

## 架构

```
┌──────────── WSL（eth0 = 172.x.x.x）─────────────┐
│  AI Agent (ZCode / Claude Code / Codex …)      │
│       │ stdio/MCP                              │
│       ▼                                        │
│  MCP Server (huashu-chrome mcp)                │
│       │ ws://127.0.0.1:8899 + token            │
│       ▼                                        │
│  Bridge Daemon  ← 绑到 0.0.0.0:8899            │
└──────────────────────┬─────────────────────────┘
                       │ ws://<WSL_IP>:8899
┌──────────────────────▼─────────────────────────┐
│  Windows                                       │
│    Chrome 扩展（侧栏里把"桥地址"配成 WSL 的 eth0 IP）│
└─────────────────────────────────────────────────┘
```

## 部署（一次配置，长期生效）

### Step 1：让桥默认绑到 0.0.0.0

```bash
echo 'export HUASHU_HOST=0.0.0.0' >> ~/.bashrc
source ~/.bashrc
```

设了这个环境变量后，**所有**启动桥的路径（`huashu-chrome bridge` 手动启动、`doctor` 自动拉起、MCP 首次调用拉起）都会绑到 `0.0.0.0`，不用每次手动传 `--host`。

不设的话默认仍是 `127.0.0.1`，行为完全和上游一致。

### Step 2：拿到 WSL 的 eth0 IP

```bash
ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'
# 比如 172.22.112.1
```

记下这个 IP，下一步要用。

### Step 3：装 Windows Chrome 扩展

1. 打开 `chrome://extensions` → 右上角开"开发者模式"
2. 点"加载已解压的扩展程序"，地址栏贴：

   ```
   \\wsl.localhost\<你的发行版名>\home\zhaiwuhao\ai\FindSkills\huashu-chrome\extension
   ```

   或者先在 WSL 里跑 `npx huashu-chrome extension --reveal` 拿路径。

3. 点工具栏 huashu-chrome 图标 → 打开侧栏
4. **设置 → 桥地址**：填入 Step 2 拿到的 WSL eth0 IP → 保存
5. 侧栏立刻尝试重连；几秒内三盏灯应该全绿

### Step 4：验证

```bash
npx huashu-chrome doctor
```

应该看到：
```
桥
  ✅ pid xxxx · 端口 8899 · 起于 xx:xx:xx
  ✅ 握手正常 · Chrome 扩展在线 (v1.3.0)
```

在 AI agent 里说"用 huashu-chrome 打开 example.com"，观察 Windows Chrome 是否真的开了对应页。

## 已知坑

### 1. WSL eth0 IP 会变

`wsl --shutdown` 之后再起，eth0 IP 可能变。每次变了要去扩展侧栏里改一次"桥地址"。

**缓解办法**（选一个）：

- **A. 切到镜像网络**：编辑 Windows 的 `%UserProfile%\.wslconfig`：
  ```ini
  [wsl2]
  networkingMode=mirrored
  ```
  然后 `wsl --shutdown` 重启 WSL。镜像模式下 Windows 和 WSL 共享 localhost，桥可以重新绑回 `127.0.0.1`，扩展也不用改地址——但会影响你现有 `wslcp` 的行为，需要测试 9222 的通路。
- **B. 写个 WSL 启动钩子**：每次 WSL 起来时自动把当前 eth0 IP 写到 `~/.huashu-chrome/host`，扩展 popup 从这里读并自动校正。当前未实现，欢迎 PR。

### 2. 扩展连不上时的诊断顺序

```bash
# 1) WSL 侧看桥绑到哪了
lsof -i :8899
# 期望：TCP *:8899 (LISTEN)
# 如果看到 127.0.0.1:8899，说明 HUASHU_HOST 没生效，重启 shell 再试

# 2) WSL 侧看桥认 host 了吗
cat ~/.huashu-chrome/bridge.json
# 期望："host": "0.0.0.0"

# 3) Windows 侧 ping WSL
# PowerShell：
ping <WSL_IP>
# 期望：能 ping 通

# 4) Windows 侧测端口
# PowerShell：
Test-NetConnection -ComputerName <WSL_IP> -Port 8899
# 期望：TcpTestSucceeded: True
```

任何一步挂了，按那一步的报错修，别往下走。

### 3. 扩展配置丢失

如果升级了扩展、重装了 Chrome、清了 `chrome.storage.local`，"桥地址"会重置回 `127.0.0.1`。重新打开侧栏填一次就行。

## 改了什么（相对上游）

```
src/cli.js
  bridge 子命令支持 --host flag 和 HUASHU_HOST env var
  help 文本说明

src/bridge.js
  startBridge({ ..., host = '127.0.0.1', ... })
  WebSocketServer({ host, ... })
  writeBridgeInfo({ ..., host, ... })

src/lib/rpc.js  ← 不用改，HUASHU_HOST 由 cli.js 读取，spawn 出的子进程会继承环境

extension/background.js
  getBridgeHost() — 从 chrome.storage.local.hcBridgeHost 读
  directConnect() 用配置 host
  onMessage 新增 'config'（查询）和 'setConfig'（写入 + 触发重连）

extension/offscreen.js
  probe(port, host, hello) — 加 host 参数
  askHost() — 从 SW 问 host
  connect() 里 host = await askHost()

extension/panel.html + panel.js
  设置区加"桥地址"输入框，保存后触发重连
```

## 与上游同步

如果想合并上游新版本，注意 host 相关的改动：
- `src/bridge.js` 的 `host` 参数和 writeBridgeInfo 落盘
- `src/cli.js` 的 `--host` flag + env var
- `extension/` 下 4 处：storage 读取、directConnect、offscreen 的 probe、panel UI

合并时保留这些。其余按上游覆盖。
