# obsidian-hermes-plugin

一个 [Hermes Agent](https://github.com/NousResearch/hermes-agent) **skill** —— 用于安装和配置
[jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin)
这款 Obsidian 社区插件,让 Obsidian 通过 gateway 的 OpenAI 兼容 API Server
(:8642) 与本地 Hermes Agent 联动。

**本仓库只包含 skill 文件**:`[productivity/obsidian-hermes-plugin/SKILL.md](./SKILL.md)`
(15.9 KB · 410 行)

## 这是什么

这个 skill 是一份 **单文件、可直接使用的 Hermes Agent 操作流程**,记录了经过
实测的端到端接通流程:

1. **启用 gateway 的 API Server**(`API_SERVER_ENABLED=true`、`API_SERVER_KEY=...`)
2. **重启 gateway**(systemd 场景,绕开 SIGTERM 自我传播守卫)
3. **验证 :8642 真的在监听**(`/health`、`/v1/models`、journalctl)
4. **安装 Obsidian 插件**(推荐走 [BRAT](https://github.com/TfTHacker/obsidian42-brat))
5. **配置插件**(baseUrl、API key、transport、working folder)
6. **用真实对话验证**

外加一份专门的 **Troubleshooting** 段,涵盖插件自身 README 没处理的 6 类问题:

- `Cannot reach the gateway`
- `Auth failed (401/403)`
- Codex 沙箱拒绝读 vault 文件(`workspace-write` 配置 +
  Windows `CreateProcessWithLogonW 1385` 企业锁机的 fallback)
- 多 profile 时端口错配
- 老 gateway 上 `/v1/models` 返回空
- chat footer 显示真实 model id 为空(Hermes home 路径)

## 安装

把 `SKILL.md` 放到你的 Hermes Agent skills 目录:

```bash
# Linux / macOS / WSL
mkdir -p ~/.hermes/skills/productivity/obsidian-hermes-plugin
curl -fsSL https://raw.githubusercontent.com/leonluo2008-ops/obsidian-hermes-plugin/main/SKILL.md \
  -o ~/.hermes/skills/productivity/obsidian-hermes-plugin/SKILL.md
```

或者通过 Hermes CLI(走 registry 路径):

```bash
hermes skills install https://github.com/leonluo2008-ops/obsidian-hermes-plugin
```

然后 **重启 Hermes 会话**(或在 session 里跑 `/reload-skills`),让新 skill 加载。

## 触发词(避免误触发)

Hermes 的 skill 触发基于 description 匹配。**Obsidian 社区里有大量类似插件**
(Claudian、Obsidian Copilot、Smart Connections 等),如果只用 "Obsidian plugin" 这种
通用词触发,会跟那些插件冲突。

本 skill 的 description 明确写了 **只在用户明确提到 jsun2020/hermes-agent-obsidian-plugin
或 Hermes Agent plugin** 时才加载。**触发词**:

- 安装 Obsidian Hermes 插件
- jsun2020 插件 / 装一下 Obsidian 的 Hermes 插件
- 配置 Obsidian Hermes 插件
- Obsidian 连不上 Hermes / Obsidian 报 401
- Obsidian plugin jsun2020 / jsun2020/hermes-agent-obsidian-plugin
- Install Obsidian Hermes Agent plugin

**不会触发**(避免误加载):

- "Obsidian plugin install"(太通用)
- "Obsidian AI plugin" / "Obsidian Copilot" / "Claudian"(别的插件)
- "Chat with your notes" 类插件(Smart Connections 等)
- Hermes Agent Desktop app 本体(那只是 `hermes desktop`,跟 Obsidian 无关)

## 使用

加载本 skill 后,说下面任何一句都会触发并执行 6 步接通流程:

- "装一下 Obsidian 的 Hermes 插件"
- "Obsidian 报 Cannot reach the gateway"
- "我的 Obsidian plugin 401 unauthorized"
- "配置 jsun2020 Obsidian 插件"
- "Install Obsidian Hermes Agent plugin"

## 这不是

- **不是** Hermes plugin 本身 —— 它是给 Hermes agent 用的 `SKILL.md`(流程性知识)
- **不是** Obsidian 插件本体 —— 那是 [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin)
- **不是** Hermes Agent 的 fork —— 只依赖 gateway 的 `API_SERVER_*` 环境变量(Hermes v0.18.2+ 自带)

## 架构

```
┌─────────────────┐                    ┌──────────────────────┐
│ Obsidian vault  │                    │  Hermes Agent        │
│ + Hermes Agent  │   /v1/runs        │  gateway (systemd)   │
│   plugin        │   /v1/models      │                      │
│                 │   127.0.0.1:8642  │  + API Server        │
│   (sidebar)     │  Bearer <KEY>     │    (启用时)           │
└─────────────────┘ ──────────────────▶└──────────────────────┘
```

插件优先用 "Runs" transport(`POST /v1/runs` + `GET /v1/runs/{id}/events`,前提是
`/v1/capabilities` 通告支持);fallback 到 OpenAI 兼容的 `/v1/chat/completions`。
Hermes Agent gateway 两个都支持。

## 验证环境

| 组件 | 版本 | 日期 |
|---|---|---|
| Zbook 主机 | Windows 11 + WSL2 Ubuntu 24.04 | 2026-07-16 |
| Hermes Agent | v0.18.2 (commit `f556edc1`) | 2026-07-16 |
| Obsidian | 1.7+ | 2026-07-16 |
| Obsidian 插件(目标) | v0.9.1 | 2026-07-16 |
| Obsidian 插件仓库 | [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin) | 安装时取最新 |

## License

MIT —— 见 [LICENSE](./LICENSE)。

## 相关链接

- [jsun2020/hermes-agent-obsidian-plugin](https://github.com/jsun2020/hermes-agent-obsidian-plugin) ——
  本 skill 要装的插件本体
- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) ——
  本 skill 要连接的 gateway 来源
- [obsidian42-brat](https://github.com/TfTHacker/obsidian42-brat) ——
  从 GitHub 装插件的推荐方式