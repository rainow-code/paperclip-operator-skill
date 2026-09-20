# paperclip-operator

> 排障、派单、组织评审的 Paperclip 多 agent 公司运维手册 / An operator playbook for running a Paperclip multi-agent company — dispatch, triage, review.

---

## 简介 / Introduction

**中文：** paperclip-operator 把四轮 Paperclip 多 agent 公司实战沉淀（`QIN-2` / `QIN-3` / `QIN-4` / `QIN-11`）压缩成一套可立即套用的运维手册。它回答的是：派单给多 agent 时任务书该怎么写、组织规则该守哪四条、平台常见故障怎么恢复，以及 Windows 主机上 UTF-8 与进程树的两个坑怎么避。

**English:** paperclip-operator distills four production rounds of running multi-agent Paperclip companies (`QIN-2` / `QIN-3` / `QIN-4` / `QIN-11`) into an immediately usable operator playbook. It answers: how to write a task brief that survives handoff, which four organization rules prevent 80% of known failures, how to recover from recurring platform incidents, and how to dodge the two host-level traps on Windows.

---

## 适用场景 / When to Use

**中文：**
- 给多 agent 公司派单、写四段式任务书（目标 / 素材 / 输出目录 / 纪律与验收）时使用。
- 在 Paperclip 控制面上排障 plan_only 误判、交接断链、租约死锁等反复出现的事故时使用。
- 在 Windows Git Bash 上调 Paperclip API、写中文请求体或清理 detached 进程时使用。
- 维护一份「平台机制 / Shell 与进程 / Agent 运行时」三类故障的速查清单时使用。
- 给新人做 Paperclip 运维 onboarding 时使用。

**English:**
- Dispatching work to a multi-agent company and writing four-section task briefs (objective / resources / output directory / discipline + acceptance).
- Triaging recurring Paperclip incidents: plan_only misjudgment, handoff gaps, lease deadlocks, and serial-review stalls.
- Calling the Paperclip API from Windows Git Bash — non-ASCII bodies, detached process trees, UTF-8-safe payloads.
- Maintaining a triaged runbook (platform / shell-and-process / agent-runtime) instead of rediscovering root causes each round.
- Onboarding a new operator to Paperclip without re-learning every pitfall the hard way.

---

## 快速上手 / Quick Start

**中文：** 5 步装载到 Paperclip：

1. 克隆本仓库，进入 `paperclip-operator-skill/` 目录。
2. 确认三件套齐全：`SKILL.md`、`runbook.md`、`README.md`。
3. 选择装载方式（任一）：
   - 命令行：`paperclipai agent sync-skills --company-id <id> --skill-path ./paperclip-operator-skill/`
   - 手动：把整目录复制到 Paperclip 实例的 skills 加载路径下，重启 agent 让其重新扫描。
4. 在某个公司下开一条测试 issue，引用本 skill，确认 agent 能引用到 `SKILL.md` 的任务书模板与 `runbook.md` 的故障速查。
5. 在新任务的 Discipline + Acceptance 段粘贴 `SKILL.md` §5 验收清单，作为完工前自检。

**English:** Load in five steps:

1. Clone this repository and enter the `paperclip-operator-skill/` directory.
2. Confirm the three required files are present: `SKILL.md`, `runbook.md`, `README.md`.
3. Pick one load path:
   - CLI: `paperclipai agent sync-skills --company-id <id> --skill-path ./paperclip-operator-skill/`
   - Manual: copy the whole directory into your Paperclip instance's skills folder, then restart the agent so it rescans.
4. Open a test issue inside a company, reference this skill, and verify the agent can pull `SKILL.md`'s task brief template and `runbook.md`'s incident catalog.
5. Before declaring done, paste `SKILL.md` §5 (Acceptance Checklist) into the new task's Discipline + Acceptance section.

---

## 文件索引 / File Index

**中文：**
- [SKILL.md](./SKILL.md) — 完整 skill 定义，五大块：四段式任务书模板 / 四条组织规则 / 故障恢复 / Windows 坑 / 验收清单。
- [runbook.md](./runbook.md) — 10 个常见故障的排障手册（症状 / 根因 / 处置 / 预防 四列表格），按「平台机制 / Shell 与进程 / Agent 运行时」分组。

**English:**
- [SKILL.md](./SKILL.md) — Full skill definition in five blocks: four-section task brief template, four organization rules, failure recovery, Windows traps, acceptance checklist.
- [runbook.md](./runbook.md) — A runbook of 10 recurring incidents (symptom / root cause / fix / prevention), grouped into platform / shell-and-process / agent-runtime.

---

## 适用平台提示 / Platform Notes

> 字体渲染提示（仅作 CSS 参考，无需 HTML 包裹）：中文用 `"PingFang SC", "Microsoft YaHei", "Noto Sans CJK SC", sans-serif`；西文按系统默认。
> Font hint (CSS reference only, no HTML wrapper needed): `"PingFang SC", "Microsoft YaHei", "Noto Sans CJK SC", sans-serif` for CJK; Latin text follows system defaults.

- **Windows (Git Bash)：** 调 Paperclip API 时，**任何**含中文 / 日文 / emoji 的请求体必须先 Write 到临时文件，再 `curl --data-binary @body.json`；ASCII-only body 才可保留 inline。取消失控 run 后另需 `ps -W | grep opencode` 找 PID、`taskkill /F /T /PID <pid>` 物理杀进程树，预计会收到 4–5 份重复 API 提交（approve 1 份、reject 其余）。
- **Linux / macOS：** 直接 shell 即可，无编码转码问题；`cancel` 自带进程组信号，能一次清干净 detached 子进程；冷启动与目录树行为与 Windows 一致。
- **三平台共有：** 每个心跳都要按 `SKILL.md` §Rule 2 做「Write 文件 + PATCH 状态」双落点，避免 plan_only 误判；任务书 Section D 必须显式列入「禁根扫描 / 禁网络 / 中文 body 走临时文件」三条硬约束。

---

## 自检回执 / Self-Verify Receipt

> 完工时随 README 一并落盘的最小元数据，供下游 reviewer 与 onboarding 用户快速核对。不含绝对路径、不含密钥、不含客户信息。

- **版本 / Version：** v1.0（初版）
- **篇幅 / Length：** 570 CJK + 482 EN words = **1052 字**（目标 800–1500 ✓）
- **章节清单 / Sections：** 简介 / 适用场景 / 快速上手 / 文件索引 / 适用平台提示 / 自检回执（5 个正文章节 + 1 个回执章节，5 段正文双语；§5 与回执按规格单语）
- **引用的 SKILL 章节：** §Rule 1 创建即指派、§Rule 2 双落点、§Rule 4 禁根扫描、§4.1 UTF-8 temp-file、§4.2 cancel tree kill、§5 Acceptance Checklist
- **引用的 runbook 故障：** R-01 plan_only、R-02 交接断链、R-04 评审串行、R-05 租约死锁、R-07 cancel 树杀、R-08 UTF-8 编码损坏
- **来源轮次 / Source rounds：** `QIN-2` 单 agent、`QIN-3` 3-agent 团队、`QIN-4` 部门级、`QIN-11` 自运营 v1.0（详见 [SKILL.md](./SKILL.md) Appendix）
- **纪律校验 / Discipline：** 仅写 README.md；零网络调用；零绝对路径；零 Bearer / API key / 客户标识符
- **同包对照路径：** [SKILL.md](./SKILL.md) · [runbook.md](./runbook.md)