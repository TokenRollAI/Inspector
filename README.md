# Inspector

**Claude API 请求与 SSE 解剖台** —— 一个纯前端、零依赖的离线解析工具。

在线使用：**https://inspector.pdjjq.org**

把 Claude API（`/v1/messages`）的请求 Body 或返回的原始 SSE 流粘贴进去，可视化地看清一次请求的 context 由什么组成、一次响应的流由什么构成。所有解析都在浏览器本地完成，**不上传任何数据**。

## 截图

**请求 Body 拆解** —— 顶层参数、体量分布、system / tools 逐块展开（下图为一个 304KB 真实 Claude Code 请求体的解析结果）：

![请求 Body 拆解](docs/screenshot-request-anatomy.png)

**Context 拆解概览** —— 控制项、三大块占比条、messages 内部内容块占比、缓存断点：

![Context 拆解](docs/screenshot-context-detail.png)

**Agent loop 分组** —— 33 条消息自动切成 4 个 Turn，识别出 user 消息中的 tool_result 回填与 system-reminder 注入，并标记用户中断：

![Turn 分组](docs/screenshot-turn-grouping.png)

**工具参数树** —— 每个 tool 可切换「描述 / 参数树 / 原始 schema」，参数树展示类型、必填与说明：

![参数树](docs/screenshot-schema-tree.png)

**SSE 重组结果** —— usage 指标卡（含 cache 读写/命中率/成本估算）、stop_reason、把 delta 拼回的完整内容块：

![SSE 重组](docs/screenshot-sse-recon.png)

## 功能

### ① 解析请求 Body

粘贴请求 JSON，得到完整的 context 拆解：

- **体量分布** —— system / tools / messages 三大块占比，以及 messages 内部 `text` / `thinking` / `tool_use` / `tool_result` 各类型占比（看清"谁在吃上下文"）
- **顶层控制项** —— `thinking`、`output_config`、`context_management`、缓存断点（`cache_control`）等
- **Agent loop 分组** —— 自动识别"`role:user` 但实为 tool_result 回填"的消息，把扁平 messages 切成 **Turn（对话轮）→ Step（步骤）** 两层结构，每轮带泳道图（user › asst › tool › …）；tool_result 按 `tool_use_id` 配回发起调用的工具名，并标注成败
- **tools 三视图** —— 每个工具可切换「描述 / 参数树 / 原始 schema」，参数树展示类型、必填、enum、default、说明
- **system-reminder 识别** —— `role:"system"` 注入消息单独标记为「💉 注入」
- 消息关键字筛选、按 Turn 分组 / 平铺双视图

### ② 解析 SSE 输出

粘贴 Claude 返回的原始 SSE 文本，三个视角：

- **重组结果** —— 把 `*_delta` 拼回完整 message（文本 / thinking / tool_use 入参），展示 `usage`（含 cache 读写、5m/1h 写入细分）、**cache 命中率**、**成本估算**（按官方价格表，cache 读 0.1×、写 5m=1.25× / 1h=2× 分别计费）、`stop_reason` / `stop_details`、`context_management.applied_edits`
- **事件流** —— 每个 SSE 事件按类型上色、可展开看原始 JSON，顶部有**类型过滤 chips**（一键屏蔽 ping / delta 噪音）
- **裸 SSE** —— 原始字节流原样呈现，高亮 `event:` / `data:`

### 通用

- 所有内容**不截断**：每段长文本都有「⧉ 复制」和「⤢ 全文弹窗」
- 输入自动持久化（localStorage），刷新不丢
- 拖拽文件直接导入；`⌘/Ctrl+Enter` 快捷解析
- 解析结果可**导出 Markdown** 摘要

## 部署

这是一个 Cloudflare Worker + 静态资源项目，零构建：

```bash
npx wrangler deploy
```

`wrangler.toml` 里的自定义域名按需修改或删除（删掉 `routes` 段则使用默认的 `*.workers.dev` 域名）。

本地预览：

```bash
npx wrangler dev
# 打开 http://localhost:8787
```

也可以不用 Cloudflare —— `public/index.html` 是完全自包含的单文件，任何静态服务器（甚至 `file://` 直接打开）都能跑。

## 结构

```
inspector/
├── wrangler.toml      # Cloudflare Worker + 静态资源配置
├── src/index.js       # Worker：仅转发静态资源
└── public/
    ├── index.html     # 全部功能（解析 + 可视化），单文件，零依赖
    └── favicon.svg
```

## 说明

- token 数为粗估（CJK ≈1.6 字/tok、其余 ≈3.8 字/tok），精确计数请用 `/v1/messages/count_tokens`
- 成本估算基于内置价格表，模型价格变动时以官方 [pricing](https://platform.claude.com/docs/en/pricing) 为准
