# x-media AI 画布视频产品 - 毕业生上手指引

Created: 2026-07-29
Status: Draft

---

## 1. 我们正在做什么

x-media 是一个面向影视/短剧/广告场景的 AI 视频工作台。用户可以在无限画布上通过自然语言和素材生成可编辑的视频分镜，并在生成过程中完成槽位补全、参数校验、成本预估与任务回滚。

核心价值：

- 画布优先：从灵感到分镜在同一画布完成
- 业务可控：不是黑盒生成，而是可校验、可计费、可回滚
- 模块清晰：按业务域拆分，便于并行开发与后期扩展

---

## 2. 你将用到的参考项目

- `D:\GoWorkSpace\open-ai-canvas`：画布、后端、Agent 的工程参考
- `D:\GoWorkSpace\gogo_trade`：分域拆解风格参考

注意：业务逻辑不要照抄参考项目，仅参考架构与模块划分。

---

## 3. 文档阅读顺序

建议按以下顺序阅读模块文档：

1. `module-map.md`：总览与依赖关系
2. `ai-canvas-video-design.md`：核心流程与总体架构
3. `[00]用户模块/README.md`
4. `[01]素材模块/README.md`
5. `[02]画布模块/README.md`
6. `[03]Agent策略模块/README.md`
7. `[04]任务调度模块/README.md`
8. `[05]计费配额模块/README.md`
9. `[06]AI网关模块/README.md`
10. `[07]存储模块/README.md`
11. `[08]后端服务模块/README.md`
12. `[09]插件与技能模块/README.md`
13. `[10]管理后台模块/README.md`
14. `[11]基础设施模块/README.md`

---

## 4. 环境准备

### 4.1 必需工具

- Go 1.21+
- Node.js 18+ / Bun
- PostgreSQL 16+
- Redis 7+
- Docker & Docker Compose（可选）

### 4.2 本地启动（推荐先用 Docker）

```bash
cd D:\GoWorkSpace\open-ai-canvas
docker compose -f docker-compose.local.yml up -d --build
```

### 4.3 后端本地启动

```bash
cd D:\GoWorkSpace\open-ai-canvas\backend
go run ./cmd/server
```

### 4.4 前端本地启动

```bash
cd D:\GoWorkSpace\open-ai-canvas\web
bun install
bun run dev
```

---

## 5. 开发入口与目录映射

| 我们的模块 | 建议代码目录 | 参考项目位置 |
| -- | -- | -- |
| 用户模块 | `internal/user` | `open-ai-canvas/backend/internal` |
| 素材模块 | `internal/asset` | `open-ai-canvas/backend/internal` |
| 画布模块 | `web/src/modules/canvas` | `open-ai-canvas/web/src` |
| Agent策略模块 | `internal/agent` | `open-ai-canvas/canvas-agent` |
| 任务调度模块 | `internal/task` | `open-ai-canvas/backend/internal` |
| 计费配额模块 | `internal/quota` | 参考策略配置 |
| AI网关模块 | `internal/gateway` | `open-ai-canvas/README.md` |
| 存储模块 | `internal/storage` | OSS 直传逻辑 |
| 后端服务模块 | `cmd/server` | `open-ai-canvas/backend/cmd` |

---

## 6. 核心概念

- Spec Snapshot：当前任务的 JSON 结构化参数
- Slot：需要补全的业务/视觉字段
- Reserve：生成前预占额度
- Pipeline：任务执行步骤链
- Channel：模型渠道（LLM/视频/图像/TTS）

---

## 7. 常见问题

Q: 前端画布从哪开始看？
A: 先看 `open-ai-canvas/web/src/components/canvas`，再对照 `[02]画布模块/README.md`

Q: 后端任务队列怎么理解？
A: 先看 `open-ai-canvas/backend/internal`，再对照 `[04]任务调度模块/README.md`

Q: Agent 逻辑放哪？
A: 先看 `open-ai-canvas/canvas-agent`，再对照 `[03]Agent策略模块/README.md`

---

## 8. 下一步行动

1. 通读 `module-map.md`
2. 打开 `open-ai-canvas` 跑通基础闭环
3. 选择 assigned 模块，对照 README 开始实现

