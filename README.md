# Flow2API Host Agent

> Linux host-side companion service for Flow2API: browser login, token auto-refresh, Web UI, and systemd support.

一个运行在 **Linux 宿主机** 上的独立 companion service，用来把 Google Labs / Flow 的登录态稳定同步回 **已有的 Flow2API** 实例。

它适合：
- 你已经部署好了 `flow2api`
- 你不想依赖 Chrome 扩展
- 你希望在服务器上自动刷新 token
- 你需要 Web UI / systemd / 定时维护能力

---

## 它和 Flow2API 的关系

- **Flow2API**：主体服务，负责模型路由、token 管理、API 能力
- **Flow2API Host Agent**：宿主机侧辅助服务，负责维护浏览器登录态并把最新 session token 推回 Flow2API

也就是说：

> 如果你已经有 Flow2API，这个项目安装好以后就能直接配合使用。

它不是 Flow2API 的替代品，而是外挂式 companion service。

---

## 核心能力

- 维护 Google Labs 登录浏览器 profile
- 通过 Chrome DevTools Protocol 读取 `__Secure-next-auth.session-token`
- 调用 Flow2API 的 `/api/plugin/update-token` 自动更新 token
- 提供 Web UI（仪表盘 / 登录向导 / 配置 / 帮助）
- 支持 systemd 常驻
- 支持可选的每日维护性重启 timer

---

## 适用前提

你需要先有：

1. **一台 Linux 服务器**
2. **已部署好的 Flow2API**
3. **Google Chrome**
4. **Xvfb / noVNC**（用于服务器上的浏览器登录）

---

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/muyouzhi6/flow2api-host-agent.git
cd flow2api-host-agent
```

### 2. 安装 systemd 服务

```bash
bash install-systemd.sh
```

### 3. 编辑配置

```bash
cp agent.example.toml agent.toml
nano agent.toml
```

重点填写：

```toml
flow2api_url = "http://127.0.0.1:38000"
connection_token = "从 Flow2API 管理后台复制"
novnc_url = "http://你的服务器IP:6080/vnc.html?autoconnect=true&resize=scale&quality=6"
listen_host = "0.0.0.0"
listen_port = 38110
```

---

## Connection Token 怎么获取？

打开你的 Flow2API 管理后台：

1. 登录管理员账号
2. 进入 **设置**
3. 找到 **插件连接配置**
4. 复制 **连接 Token**

如果为空，就先生成一个再保存。

它的作用是：

> Host Agent 每次把 session token 推送回 Flow2API 时，Flow2API 用它验证“这个请求是不是你授权的来源”。

---

## 首次登录流程

### 1. 启动浏览器

```bash
systemctl start flow2api-host-agent-browser
```

### 2. 启动 Web UI

```bash
systemctl start flow2api-host-agent-ui
```

打开：

```text
http://你的服务器IP:38110/login
```

### 3. 在登录页中完成 Google 登录

- 页面内会嵌入 noVNC
- 直接在页面中操作服务器上的 Chrome
- 登录与 Google Labs 一致的账号

### 4. 手动触发一次刷新

在 Web UI 仪表盘点击：

- **立即刷新 Token**

看到成功即可。

---

## 自动运行

### 启动 daemon

```bash
systemctl start flow2api-host-agent
```

### 开机自启

安装脚本会自动 `enable`：

- `flow2api-host-agent`
- `flow2api-host-agent-ui`
- `flow2api-host-agent-browser`

---

## 可选：每日维护性重启

如果你希望进一步降低 Chrome 长时间运行的累积状态问题，可以开启：

```bash
systemctl enable --now flow2api-host-agent-daily-restart.timer
```

默认计划时间：

- **每天 04:30 UTC**

它会重启：
- browser
- daemon
- ui

这是一个偏稳妥的长期运行策略。

---

## Web UI 页面说明

- **仪表盘**：看最近一次刷新结果和当前状态
- **登录向导**：首次使用 / 登录失效时用
- **配置**：修改 Flow2API 地址、Connection Token、noVNC 地址等
- **帮助 / 原理 / 稳定性**：解释它怎么工作、会不会越跑越重、为什么建议维护性重启

---

## 注意事项

### 不要把 noVNC 地址写成 localhost

如果你是用手机或另一台设备访问登录页，`novnc_url` 不能写成：

```toml
novnc_url = "http://localhost:6080/..."
```

因为这会让浏览器去访问**你自己设备的 localhost**，不是服务器。

正确方式应该是：

```toml
novnc_url = "http://你的服务器IP:6080/vnc.html?autoconnect=true&resize=scale&quality=6"
```

### 主要资源开销来自 Chrome

- Python daemon：轻
- Web UI：轻
- 大头是 Chrome / renderer / GPU 相关进程

所以如果长期运行，建议保留每日维护性重启方案。

---

## 目录结构

- `scripts/`：CLI 与核心逻辑
- `web/`：Web UI
- `systemd/`：systemd service / timer 模板
- `docs/`：设计与部署文档
- `install-systemd.sh`：一键安装脚本

---

## License

待补充。
