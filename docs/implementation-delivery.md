# 实施交付案例包（Implementation Delivery Package）

> 项目：**AI Business Workflow Automation System**（n8n 客户需求自动处理）
> 角色定位：这是一个"需求 → 流程编排 → 部署 → 测试 → 交付文档"完整闭环的实施交付案例，用于展示**实施工程师**能力。
> 配套文档：部署 [README_CN.md](../README_CN.md) · 节点设计 [nodes.md](nodes.md) · 测试记录 [test-report.md](test-report.md) · 安装教程 [install-tutorial.md](install-tutorial.md)

---

## 1. 交付物清单

| 交付物 | 位置 | 说明 |
|---|---|---|
| 主工作流定义 | `workflows/customer-inbox-ai-automation.json` | 9 阶段 / 13 节点，自动导入即用 |
| 错误处理工作流 | `workflows/error-handler.json` | 异常自动飞书告警（已启用并绑定主流程） |
| 一键部署编排 | `docker-compose.yml` | n8n 主服务 + n8n-import 一次性导入（共享数据卷） |
| 环境变量模板 | `.env.example` | 国产模型 / 飞书 / RAG 全部可配，无重复定义 |
| 模拟数据 | `simulation/knowledge-base.md`、`simulation/sample-tickets.json` | 25 条样例工单 + 知识库，演示可离线跑 |
| 前端演示 | `index.html` / `demo/index.html` / `form/index.html` | 零 CDN、零第三方 JS，可托管到 GitHub Pages |
| 验收证据 | 本文第 5 节 + [test-report.md](test-report.md) | 每条验收都有方法、状态、责任人（待实跑项明确声明） |

---

## 2. 架构与流程（文字版链路图）

**主工作流（客户需求自动处理）**

```
① Webhook 接收 ──► ② LLM 需求分析 ──► ③ AI 智能分类 ──► ④ 知识库检索(RAG)
   (统一入口)       (温度0.1: intent/    (7类预置)        (可选,未配置自动跳过)
                    sentiment/urgency/
                    expectedAction)
        │
        ▼
⑤ 自动生成摘要 ──► ⑥ 结构化工单(Set) ──► ⑦ SLA 矩阵 ──► ⑧ 飞书多维表格 ──► ⑨ 飞书卡片通知
   (温度0.4,≤60字)   (TK-日期-流水)       (28组合:P1~P4   (retryOnFail)      (标题带[SLA]徽章,
                                          +责任团队)                         红色critical/橙色high)

受理回执(respondToWebhook)：处理完成后立即把 ticket_id 回给提交方
```

**错误处理工作流（error-handler.json，2 节点）**

```
Error Trigger ──► 飞书告警卡片（工作流名 + 出错节点 + 错误信息 + 时间 → 管理员）
```

**关键设计决策**（为什么这样拆，面试可讲）
- 分析 / 分类 / 摘要三段式而非一个大 Prompt：分类规则独立可调、出错可定位、可单独换模型（详见 nodes.md）
- 分温度策略：分析/分类 0.1（稳定）、摘要 0.4（自然）
- 国产模型直通：节点参数内嵌 `$env.LLM_MODEL` 等表达式，切换模型只改 `.env` 三行，不改节点
- 导入与数据卷：n8n 官方不自动扫描目录，`n8n-import` 显式 import 并与主服务共享 `n8n_data` 卷，避免"导入了但主服务看不到"

---

## 3. 环境与依赖

**前置条件**：Docker + Docker Compose v2.24+；任一国产大模型 API Key；飞书自建应用凭据（可选，用于通知/表格）。

| 变量 | 用途 | 默认/示例 |
|---|---|---|
| `LLM_MODEL` | 模型名（工作流实际读取） | `deepseek-chat` |
| `OPENAI_BASE_URL` | 模型网关（工作流实际读取） | `https://api.deepseek.com/v1` |
| `OPENAI_API_KEY` | 模型密钥（工作流实际读取） | `sk-...`（占位） |
| `MODEL_PROVIDER` | 供应商标识（测试脚本用） | `deepseek` |
| `FEISHU_APP_ID/SECRET` | 飞书应用 | `cli_...` |
| `FEISHU_BITABLE_APP_TOKEN/TABLE_ID` | 多维表格写入目标 | `bascn...`/`tbl...` |
| `FEISHU_OWNER_USER_ID` | 通知默认负责人 | `ou_...` |
| `KB_SEARCH_URL` / `KB_TOP_K` | RAG 检索（可选） | 留空自动跳过 / 3 |
| `WEBHOOK_URL` | 公网 Webhook 回调地址 | `http://localhost:5678/` |

> 模型切换只改 `LLM_MODEL` / `OPENAI_BASE_URL` / `OPENAI_API_KEY` + `MODEL_PROVIDER`（DeepSeek / 通义 / 智谱 / Kimi / OpenAI 兼容网关均支持，详见 README_CN 第 4 节）。

---

## 4. 启动 / 停止 / 导入

```bash
# 启动（国内先配镜像加速，见 README_CN 3.0 节）
cp .env.example .env          # 填模型 Key + 飞书凭据
docker compose up -d          # n8n 起 → 健康检查 → n8n-import 自动导入工作流
# 打开 http://localhost:5678，在 UI 中激活主工作流

# 检查导入是否成功
docker compose logs n8n-import   # 看到 "workflows imported"

# 停止
docker compose down

# 数据重置（重导一遍演示数据）
docker compose down && docker volume rm <项目名>_n8n_data && docker compose up -d
```

**本次修复的导入链路说明（AI-01）**：`n8n-import` 与主服务共享命名卷 `n8n_data`，`import:workflow --input=/workflows` 写入的位置就是主服务读取的位置；`list:workflow --onlyId` 判空避免重复 up 时重复导入。

**错误工作流绑定（AI-03）**：`error-handler.json` 固定工作流 id（`errorHandlerMain`）并 `active: true`；主工作流 `settings.errorWorkflow: errorHandlerMain` 自动绑定——任意节点执行失败即触发告警，无需手动配置。

---

## 5. 功能验收清单

> 状态图例：✅ = 本机/静态已验证（方法见 test-report.md 与本文件"验证记录"）｜⏳ = 配置已就绪，需真实环境（Docker / API Key / 飞书凭据）实跑

### 正常路径（8 项）

| # | 验收项 | 预期结果 | 状态 |
|---|---|---|---|
| N1 | 表单/Webhook 提交 | 收到请求并返回受理回执（`code:0, ticket_id:TK-...`） | ✅ 2026-10-10 演示页实测 9 节点全通 |
| N2 | LLM 需求分析 | 输出 intent / sentiment / urgency / expectedAction 四字段 | ⏳ 填 Key 后实跑 |
| N3 | AI 智能分类 | 从 7 个预置分类选出 1 个 | ⏳ 同上 |
| N4 | RAG 检索增强 | 命中时摘要带知识库依据；未配置 `KB_SEARCH_URL` 自动跳过 | ✅ 容错逻辑静态确认 |
| N5 | 自动摘要 | ≤ 60 字、含分类与关键信息 | ⏳ 实跑 |
| N6 | SLA 矩阵 | 28 组合映射正确（投诉+高/紧急→P1；技术支持+critical→P1） | ✅ Code 节点语法通过 + 演示页实测 P1/30min |
| N7 | 飞书多维表格写入 | 工单实时新增一行（含 SLA level/team） | ⏳ 需飞书凭据 |
| N8 | 飞书卡片通知 | 标题带 `[P1]` 等徽章，红色/橙色/蓝色分级 | ✅ 2026-10-10 卡片渲染实测；实跑需飞书凭据 |

### 失败路径（5 项）

| # | 故障注入 | 预期行为 | 状态 |
|---|---|---|---|
| F1 | 上游 LLM 调用失败 | Error Trigger 触发 error-handler，管理员收到告警卡片 | ✅ 绑定已修复+静态确认；运行 ⏳ |
| F2 | 飞书写入瞬时失败 | 节点 `retryOnFail` 自动重试 | ✅ 配置静态确认 |
| F3 | RAG 服务不可达 | 检索节点跳过，摘要仍生成（不带引用） | ✅ 容错逻辑静态确认 |
| F4 | 重复 `docker compose up` | `list:workflow` 判空，跳过重复导入 | ✅ 脚本逻辑静态确认 |
| F5 | 导入后主服务看不到工作流 | 共享 `n8n_data` 卷修复（修复前：匿名卷导致不可见） | ✅ compose 配置验证通过；整体 ⏳ |

### 验证记录（2026-10-27 修复回归）

| 项 | 方法 | 结果 |
|---|---|---|
| 工作流 JSON 合法性 + 绑定 | `json.load()` 解析两个 workflow | ✅ error-handler `active=true, id=errorHandlerMain`；主流程 `errorWorkflow=errorHandlerMain` |
| docker-compose YAML + 卷 | `yaml.safe_load()` | ✅ services=n8n/n8n-import；`n8n-import` volumes 含 `n8n_data:/home/node/.n8n` |
| .env.example 变量唯一性 | 文本扫描 | ✅ `OPENAI_BASE_URL=` / `OPENAI_API_KEY=` 各仅 1 处定义（修复重复覆盖 bug） |
| 完整回归基线 | [test-report.md](test-report.md) 2026-10-10 版 | ✅ 13 项静态/本地/浏览器实测通过 |

---

## 6. 已知问题与限制（诚实声明）

1. **Docker 运行未实跑**：开发机无 Docker，`docker compose up` 的容器级验证待用户在目标机执行（见 test-report.md 第 2 节）。
2. **真实模型未调用**：没有 API Key，`test_ai_provider.py` 只验证了配置解析与切换逻辑，真实调用待填 Key 后执行。
3. **飞书集成未实跑**：无飞书应用凭据；表格写入与卡片通知的端到端待配置后验证。
4. **中国大陆网络未实测**：镜像加速、DeepSeek/通义等网关延迟待国内环境确认（已提供全部国内直连配置）。
5. **多进程并发**：本案例为单实例 n8n；若多实例共享工作流数据库，需数据库级并发控制（当前设计不涉及）。
6. **模拟数据边界**：`simulation/` 下数据仅用于演示，不代表真实业务数据；生产接入需替换为真实工单与知识库。

---

## 7. 3~5 分钟演示脚本（面试用）

**开场（30 秒）**：这是一个基于 n8n 的客户需求自动处理系统：客户留言进来，模型分析情绪与紧急度，按 28 种组合路由到负责人，全程写入飞书多维表格并通知卡片，出错自动告警。设计原则是"可维护性优先"——分类规则、模型、SLA 口径都能独立调整。

**演示步骤（3 分钟，假设已配好 Key 与飞书）**
1. `docker compose up -d`，打开 `http://localhost:5678`，展示 13 节点工作流已自动导入、error-handler 已绑定（30 秒）
2. 激活主工作流，用 curl 或表单提交一条"我要退款，今天必须处理"的留言（20 秒）
3. 现场展示：curl 秒回 `ticket_id`；飞书表格多一行；负责人收到带 `[P1]` 徽章的红色卡片（60 秒）
4. 切模型：改 `.env` 三行从 DeepSeek 切到通义，重启，再次提交（30 秒）——证明"换模型不改节点"
5. 故障演示：故意停掉模型 Key 触发错误，展示管理员收到错误告警卡片（40 秒）

**收尾（30 秒）**：指出交付包里有完整验收清单与测试报告；未实跑项（Docker/真实 Key/飞书）如实说明并给出验证方法——展示的是"会交付、会验证、会讲边界"而不是"全都跑过"。

---

## 8. 模拟数据与演示边界

- `simulation/knowledge-base.md`：模拟企业知识库（网络故障 / Windows 处理 / VPN / 邮箱四类），导入向量库后即可演示 RAG 增强。
- `simulation/sample-tickets.json`：25 条样例工单，覆盖 7 分类 × 4 紧急度，供 SLA 矩阵与统计演示。
- 前端演示（`index.html` 等）：在浏览器内本地模拟完整流水线（含卡片渲染），无后端也可看效果——**仅用于演示，不代表真实履约**。
- 上线前替换项：真实知识库、真实工单表、真实负责人列表、真实通知渠道。

---

## 9. 后续迭代建议

- 给飞书 `tenant_access_token` 增加自动刷新节点（当前建议手动维护或前置 HTTP 节点刷新）。
- 把 SLA 超时考核做成定时任务（n8n Cron）生成日报。
- 知识库检索接入带权限的检索服务（如 Qdrant + API Key），避免直接暴露内网检索。
- 为两个 LLM 节点增加失败降级（如分类失败回退规则关键词分类）。
