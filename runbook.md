# runbook.md — Paperclip Operator 排障手册

> 适用对象：Paperclip 公司运维者（CEO / 调度员 / 评审官）
> 数据来源：四轮实战复盘 `QIN-2` / `QIN-3` / `QIN-4` / `QIN-11`（详见底部）
> 配套阅读：`SKILL.md` §3（故障恢复）、§4（Windows 坑）

每条故障给出：**症状 | 根因 | 处置 | 预防**。处置命令按 Windows Git Bash 路径编写；Linux/Mac 仅路径风格不同，命令本身一致。

---

## 一、平台机制类（来自控制面行为）

### R-01 plan_only 误判重置任务状态

| 字段 | 内容 |
|---|---|
| 症状 | 心跳内已完成 Write + 落盘，下一轮 `GET /api/issues/{id}` 发现 status 退化为 `Todo`；run transcript 无文件写入证据 |
| 根因 | opencode 适配器的 liveness 判定只扫描 transcript 抓取文件路径；上一轮若预算/输出已耗尽，Write 工具调用未被记录即被读为「无操作」（`QIN-2` §四.1） |
| 处置 | 同一心跳内**先** Write 文件**再** PATCH 控制面状态（双落点）；若已被重置，重跑产出并立刻 PATCH `done`，不留跨心跳空隙 |
| 预防 | 见 `SKILL.md` §Rule 2 双落点；任务书验收章明确要求「文件 + 状态」配对落盘 |

### R-02 交接断链：子任务落进组织缝隙

| 字段 | 内容 |
|---|---|
| 症状 | 创建子任务后，依赖解除或父任务完成，子任务长期停在 `todo`；creator 心跳和 assignee 心跳都「队列无待办」被跳过（`QIN-3` §三.1 P0） |
| 根因 | 依赖唤醒只认 `assigneeAgentId`；空 assignee 永远不会触发 wake；creator 自身的空队列启发式也不为子任务补发唤醒 |
| 处置 | 重发创建调用，补 `assigneeAgentId`（或人类任务补 `assigneeUserId` 并置 `in_review`）；如已存在则 PATCH 指派，**不要**用 free-text「@X 请接手」 |
| 预防 | `SKILL.md` §Rule 1 创建即指派；任务书模板 Section D 列入「子任务必须有 owner」硬验收项 |

### R-03 订阅型接入预算计量失效

| 字段 | 内容 |
|---|---|
| 症状 | 客户用 Claude Max / ChatGPT Plus / coding plan 等订阅 key 接入；`costCents` 恒为 0，token 计数正常但 `billed_cents=0`，预算熔断永不触发（`QIN-3` §三.3 P1） |
| 根因 | 自定义 OpenAI 兼容端点无公开价格表；Paperclip 计费层无法换算 `costCents`；`billed_cents` 维度的硬停形同虚设 |
| 处置 | 在交付层加 token→美元估算（如 `1M tokens ≈ $X` at retail）；按估算值做软熔断；向客户明示「估算预算，非计量预算」 |
| 预防 | 签 L3 维护合同时**不承诺**计量预算硬停；只承诺估算软上限；SOW 条款须与运行时同步对齐 |

### R-04 评审官单次 run 只消化一个 in_review

| 字段 | 内容 |
|---|---|
| 症状 | 同时存在多个 `in_review` 任务；评审官 wake 一次只完成第一个，后续任务在其 run 活跃期被 defer，run 结束后无人续跑（`QIN-4` §二 #11） |
| 根因 | 评审官按串行处理 review；同 assignee 多 in_review 不会自动 stack；run 退出后依赖唤醒不会再次触发同 assignee |
| 处置 | 董事会（或父任务 owner）逐个 PATCH `done` 点火（`QIN-4` 实验中 3 次评审均手动点火完成）；不要期待评审官自助 drain |
| 预防 | DAG 设计阶段即预知串行成本；需快速评审时增设多个评审员（不是同一 agent 复用） |

### R-05 租约死锁：cancel 后所有后续唤醒被无限 defer

| 字段 | 内容 |
|---|---|
| 症状 | 某 run 被 cancel 后，对应 issue 长期 `in_progress` 无进展；server log 出现 `previous execution has not released its environment lease`；后续 wake 一律 skip（`QIN-4` §二 #9 P0） |
| 根因 | cancel 后租约 `cleanup_status='failed'` 且无重试无升级；`getConversationViewPostBlocker` 永远视为活跃；补丁 `ce8d89233` 加 30 分钟宽限（`PAPERCLIP_FAILED_CLEANUP_LEASE_GRACE_MS` 可调） |
| 处置 | **有补丁**：等 30 分钟宽限；若仍锁，`agent_wakeup_requests` 表直查定位队列状态（`QIN-4` 用 `node+postgres` 连 54329）；Windows 上手动清 PID 记录（PID 复用 + `startedAt` 读取失败保守判活）。**无补丁**：必须直查租约表并手工释放 |
| 预防 | 非必要不 cancel；cancel 后立即设置 `monitorNextCheckAt` 在 35 分钟后的 monitor，确保宽限到期后有 wake 跟进 |

---

## 二、Shell 与进程类（来自主机层行为）

### R-06 无界 shell 命令烧穿 run（`find /`）

| 字段 | 内容 |
|---|---|
| 症状 | 单条 `find /` 或 `ls /` 让 run 卡死 30+ 分钟，烧 3 小时 CPU；agent 其余 wake 全部排队等待（`QIN-4` §二 #7） |
| 根因 | opencode 子进程默认无命令超时/路径围栏；根目录递归遇大目录直接失控 |
| 处置 | `ps -W` 找到失控 PID，`taskkill /F /T /PID <pid>`；必要时强制取消该 run；不要尝试给原命令发信号 |
| 预防 | 任务书 Section D 列入「禁根扫描」令（`SKILL.md` §Rule 4）；所有 shell 命令显式指定根路径与边界；优先用 grep/glob 替代 find |

### R-07 cancel 杀不透 detached 进程树

| 字段 | 内容 |
|---|---|
| 症状 | cancel 后 Paperclip run 状态 `cancelled`，但底层 opencode 子进程及孙进程仍在运行；临死前还可能提交若干 API 调用（`QIN-4` 案例中产出了 4 份乱码重复招聘申请，`§二 #8`） |
| 根因 | Linux 信号能打进程组；Windows 要靠 job object 或 `taskkill /T` 走树；Paperclip 适配器默认不开 job object |
| 处置 | `ps -W | grep opencode` 找 PID → `taskkill /F /T /PID <pid>`；预计会收到 4–5 份重复 API 提交，approve 1 份、reject 其余（agent 通常能自纠） |
| 预防 | 减少 cancel 用量；确需 cancel 时先 `taskkill` 再走 API；Windows 跑长时间任务优先放进 job object |

### R-08 UTF-8 编码损坏：curl 中文请求体变 `?????`

| 字段 | 内容 |
|---|---|
| 症状 | `curl -d '{"title":"中文..."}'` 提交后服务端收到 `?????`；任务标题/留言乱码但 HTTP 返回 2xx；文件落盘（Write 工具）不受影响（`QIN-4` §二 #10） |
| 根因 | Windows Git Bash 子 shell 默认 GBK / cp936；非 ASCII 字节经 console transcoding 变成 `?`；`curl -d '...'` 参数路径走 shell 而 Write 工具直写 UTF-8 |
| 处置 | **不要重发**——重发仍会损坏。改用临时文件法：Write 工具写 `body.json` → `curl --data-binary @body.json`；ASCII-only body 可保留 inline |
| 预防 | 任务书 Section D 明示「中文 API 请求体走临时文件」；含非 ASCII 的 body 一律走文件路径 |

---

## 三、Agent 运行时类（来自模型/适配器行为）

### R-09 冷启动超时抬高单轮延迟

| 字段 | 内容 |
|---|---|
| 症状 | 每次心跳 spawn 新 opencode 实例；前 15–30 秒都在拉 `models.dev`，单轮 wall-clock 偏高（`QIN-2` §四.3） |
| 根因 | 适配器未本地缓存 catalog；models.dev 远端拉取受网络抖动影响 |
| 处置 | 预算估算时给冷启动留 30s buffer；不要把冷启动时长计入实际工作时长 |
| 预防 | 上游级：考虑离线模式或本地缓存 catalog；运维级：错峰调度避免多 agent 同时冷启动 |

### R-10 think 块泄漏进任务留言

| 字段 | 内容 |
|---|---|
| 症状 | agent 思考过程（`<think>...</think>`）原样进入 issue 评论流；多轮对话后留言信噪比下降（`QIN-2` §四.2、`QIN-3` §三.6 P3，已知复发） |
| 根因 | M3 模型 think 块未被上游过滤；适配器直传原始输出到评论 API |
| 处置 | 删掉泄漏的 think 段，必要时重发精炼后的留言；不要依赖模型自律 |
| 预防 | 任务书 Section D 要求「留言先想后贴」；要求上游加 think-block 过滤；评论模板内嵌「非思考正文」自检提示 |

---

## 四、命令速查（按场景分组）

### 强制取消 + 进程清理（Windows）

```bash
# 1. 找失控 PID
ps -W | grep -E "opencode|node"
# 2. 物理杀进程树
taskkill /F /T /PID <pid>
# 3. 清 API 重复（预计 4–5 份）
#    在 UI/issue 列表中 approve 1、reject 其余
```

### UTF-8 安全 API 调用（Windows）

```bash
# 1. Write 工具写 body.json（UTF-8 安全）
# 2. 提交文件
curl -s -X POST \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @body.json \
  "$PAPERCLIP_API_BASE/api/companies/$PAPERCLIP_COMPANY_ID/issues"
```

### 租约排障（无论补丁是否生效）

```bash
# 直查 wakeup 队列（QIN-4 实测路径）
node -e "const{Client}=require('pg');const c=new Client{port:54329}...;c.query('SELECT ...').then(r=>console.log(r.rows))"
# 识别 stuck lease → 手工 release → 验证下一轮 wake 恢复
```

### 验证 PATCH 真的成功（不要靠 exit code）

```bash
# 用 helper；它检查 HTTP 状态、重试连接失败、确认 echo 回 status
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status done <<'MD'
Done — file written to <path>。
MD
# 手写 curl 也行，但必须 -w '%{http_code}' 并检查 echo
```

---

## 五、数据来源

- `QIN-2`：单兵内容情报部（22 min）— `BOARD-REVIEW.md` §四 → R-01、R-09、R-10
- `QIN-3`：3 人 EN 翻译扩张包（85 min）— `BOARD-REVIEW-2.md` §三 → R-02、R-03、R-04、R-10
- `QIN-4`：5 人 AI 客服部门级（6.8h）— `BOARD-REVIEW-3.md` §二 → R-05、R-06、R-07、R-08、R-04
- `QIN-11`：Ops v1.0 自运营（85 min 零故障）— `BOARD-REVIEW-4.md` §三补丁回归验证（10/10 通过，零复发）

> 维护提示：本手册的故障条目只增不减。新发现的实战问题请追加到对应分组（平台机制 / Shell 与进程 / Agent 运行时），并在「数据来源」一行补 `QIN-X` 引用。