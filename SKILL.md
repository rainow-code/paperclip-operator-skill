---
name: paperclip-operator
description: >
  Operate a multi-agent Paperclip company end-to-end: compose task briefs, route work, run review chains, and recover from common platform failures. Use when 运维多 agent 公司、排障、派单、创建即指派、四段式任务书、交接断链、租约死锁, or when the user is planning, scaling, or debugging an agent company on Paperclip. Derived from four production rounds (QIN-2 single agent, QIN-3 small team, QIN-4 department, QIN-11 self-operating).
license: MIT
---

# paperclip-operator

A field-tested playbook for running a Paperclip agent company. Every rule and countermeasure below was extracted from four real production rounds (`QIN-2` single agent, `QIN-3` three-agent team, `QIN-4` department, `QIN-11` self-operating). It is intentionally minimal — a working operator toolkit, not a treatise.

The skill is organized as five blocks:

1. **Four-section task brief** — the only template you should use to dispatch work
2. **Organization rules** — the four rules that prevent 80% of known failures
3. **Failure recovery** — symptom, root cause, fix, prevention for the recurring incidents
4. **Windows-specific traps** — the two host-level pitfalls on Windows Git Bash
5. **Acceptance checklist** — what "done" looks like for a Paperclip skill package

---

## 1. The four-section task brief

Every dispatched task in a Paperclip company — whether assigned to a person, a freshly-hired agent, or yourself — must be written in exactly four sections. The shape was validated across all four rounds and is the single biggest reason projects stayed unblocked.

### Section A — Objective

One paragraph. What concrete artifact will exist when this task is done, and why. Anchor the goal to a tangible deliverable, not to an activity. A reader should be able to point to the file(s) on completion and say "yes, that is the thing."

**Example (QIN-4, AI customer-service KB delivery):** *"把公司四轮实战沉淀（QIN-2/3/4/11）转化为一份对外可发布的 skill 包 `paperclip-operator-skill/`。本任务覆盖三个文件：SKILL.md / runbook.md / README.md。"* (verbatim from the issue description)

### Section B — Read-only resources

A flat list of paths or QIN-X identifiers the assignee may read. State the read-only nature explicitly. This is the source of truth and the anti-fabrication guard: an agent that can only read what you list cannot invent sources.

**Example (QIN-4):** *"四份实验复盘：`BOARD-REVIEW.md` (QIN-2) / `BOARD-REVIEW-2.md` (QIN-3) / `BOARD-REVIEW-3.md` (QIN-4) / `BOARD-REVIEW-4.md` (QIN-11)。格式范本：`ref/EXAMPLE-paperclip-official.md`、`ref/EXAMPLE-community-style.md`。"

### Section C — Output directory

Exactly one directory, named in the brief. No "you can also write to…" hedging. Cross-department contamination — the most common discipline violation — starts here, so the rule is one brief → one directory.

**Example (QIN-18.1, this skill package):** *"`workspace/<output-dir>/paperclip-operator-skill/`，含 SKILL.md / runbook.md / README.md。"*

### Section D — Discipline + acceptance

Two sub-lists. Discipline: the prohibitions (no network, no root scan, no cross-directory writes, no absolute paths in deliverables, no customer info). Acceptance: a checklist the assignee self-verifies against before PATCH-ing `done`. The acceptance list is also the board's audit hook.

**Example (QIN-18.1 discipline):**
- 只写 `workspace/<output-dir>/paperclip-operator-skill/` — 不得越界写其他目录
- `ref/` 目录只读 — 不得修改或复制范本文件
- 禁止网络 — 不得调用任何外部 URL/CDN/字体
- 禁止根目录扫描 — 禁止 `find /`、`ls /` 等无界命令
- 中文 API 请求体走临时文件（如需调 Paperclip API）— Windows Git Bash 路径
- frontmatter 严格合规 — name/description/license 三件套齐全

**Why four sections, not three or five.** QIN-2 (single agent) was written in three sections and worked. QIN-3 (three-agent team) reused the same template and immediately exposed the "交接断链" failure mode — work fell into the seam between creator and assignee because the brief did not pin down ownership. QIN-4 added the explicit Discipline+Acceptance section and the handoff gap did not recur (BOARD-REVIEW-3 explicitly notes "交接断链没有复发"). QIN-11 ran zero troubleshooting interventions using the same shape. The four-section format is therefore not aesthetic — it is the smallest structure that closes the seam.

---

## 2. Organization rules (the four rules)

These four rules, observed consistently, prevent the four most expensive failure modes seen in production. They are not optional.

### Rule 1 — 创建即指派 (Create-then-assign, atomically)

Every task creation must land in `todo` with an `assigneeAgentId` set in the same call. Never create a task and "let someone pick it up later." This rule exists because of the QIN-3 P0 finding (BOARD-REVIEW-2 §三.1): dependency wake only fires for the named assignee, and the creator's heartbeat skips when its own queue is empty. A child task with no assignee sits in the seam forever — both sides sleeping, no progress.

**Execution detail:** in the issue-creation body, set both `assigneeAgentId` and (if blocked) `blockedByIssueIds` at the same time. The compound create-call is the unit of correctness; do not split it into create-then-update.

**Anti-pattern (do not do):** create subtask → reply on parent with "I created the subtask, X can pick it up" → close own heartbeat. This is the exact seam-fall pattern QIN-3 hit.

### Rule 2 — 完工双落点 (Double-landing on completion)

Every finished task must leave durable evidence in **two** places before the assignee PATCHes `done`:

1. The deliverable file(s) at the path declared in Section C of the brief.
2. The issue's status PATCH echoing the completion (and, when relevant, a delivery comment containing the absolute output path).

This rule exists because of the QIN-2 plan_only misjudgment (BOARD-REVIEW §四.1): the opencode adapter's liveness check could not detect file-on-disk evidence, and the first two heartbeats were misclassified as plan_only and reset. The fix was the "Write 文件 + PATCH 控制面状态" double mutation — write the file **and** update the status in the same heartbeat. After this discipline was adopted, QIN-2 finished in 22 minutes and QIN-11 finished with zero board troubleshooting interventions.

**Why two mutations.** One is not enough. A file write without a status update is invisible to liveness. A status update without a file write is invisible to reviewers. The pair is the unit.

### Rule 3 — 评审串行链 (Serial review chain)

When a task needs a reviewer, exactly one reviewer must own the verdict at a time. If multiple tasks are ready for review simultaneously, the reviewer handles them one per run, in order. This rule is the QIN-4 #11 finding (BOARD-REVIEW-3 §二): the reviewer is woken, processes one `in_review`, and subsequent review-ready tasks queued behind it get deferred until the next reviewer run — they do not auto-stack. Plan for serial, not parallel, review.

**Execution detail:** in the task DAG, do not assume the reviewer can drain N reviews in one heartbeat. If you need faster turnaround, you need multiple reviewers — but that is a different project structure, not a tweak to the brief.

### Rule 4 — 禁根扫描 (No root scanning)

Task briefs must contain an explicit prohibition on unbounded shell commands: `find /`, `ls /`, recursive walks of the filesystem root, and any shell loop with no path fence. This is the QIN-4 #7 finding (BOARD-REVIEW-3 §二): a runaway `find /` ran for 46 minutes and burned 3 hours of CPU on a single run, blocking all other wakeups on the agent.

**Execution detail:** include a line in Section D of the brief: *"禁止无界 shell 命令（`find /`、`ls /`、无 path 围栏的递归）。所有 shell 命令必须显式指定根路径与边界。"*

**Why the rule is in the brief, not just in agent training.** The brief is the durable artifact the board audits. A rule only in the agent's prompt is unfalsifiable; a rule in every brief is a checklist item the board can grep.

---

## 3. Failure recovery (the recurring incidents)

These four failures recur across rounds. Each entry: symptom, root cause, fix, prevention. Detailed symptom-by-symptom recovery commands live in `runbook.md`.

### 3.1 plan_only 误判 (QIN-2, BOARD-REVIEW §四.1)

**Symptom.** After writing files to disk, the issue status flips back from `done` to `Todo`. The run transcript shows no record of the file write. The agent wastes 2+ heartbeats retrying before realizing.

**Root cause.** The opencode adapter's plan_only detector reads the run transcript for tool-call evidence. A successful Write tool call leaves a transcript entry; but if the agent's heartbeat budget or output was exhausted before the transcript was flushed, the detector sees "no work done" and resets.

**Fix.** Apply the double-landing rule (§Rule 2). Write the file AND PATCH the issue status in the same heartbeat. Do not rely on the file alone, and do not rely on the status alone.

**Prevention.** Always plan the heartbeat so the Write and the PATCH land in the same tool-call sequence. If budget is tight, write the smallest deliverable first, PATCH, then enrich.

### 3.2 交接断链 / Handoff gap (QIN-3 P0, BOARD-REVIEW-2 §三.1)

**Symptom.** A task created with no assignee (or assignee outside the org) sits in `todo` indefinitely. Both the creator's and the assignee's heartbeats fire on schedule but find no actionable work; the task is "in the seam."

**Root cause.** Dependency wake keys on the named assignee; if there is no assignee, no wake fires. The creator's queue-empty heuristic does not retry creating wakes for unassigned children.

**Fix.** Re-issue the create-call with `assigneeAgentId` set. If the assignee is human, use `assigneeUserId` and PATCH the status to `in_review`.

**Prevention.** §Rule 1 (创建即指派). Verify `assigneeAgentId` is set immediately after issue creation; if not, fix the same heartbeat.

### 3.3 租约死锁 / Lease deadlock (QIN-4 #9 P0, BOARD-REVIEW-3 §二)

**Symptom.** After a run is cancelled, no subsequent wakeups fire for the affected issue. The server log shows `previous execution has not released its environment lease`. Issue status remains `in_progress` with no progress for hours.

**Root cause.** A cancelled run leaves the environment lease in `cleanup_status='failed'` with no retry and no escalation. Subsequent wakeups see the lease as active and defer/skip themselves. (Patched in `ce8d89233` with a 30-minute grace period; earlier rounds did not have the patch and hit this.)

**Fix (post-patch).** Wait for the 30-minute grace; if still locked, manually clear the PID record from the agent's run-state table (the patch requires this on Windows because PID recycling + `startedAt` read failures make the conservative "is alive" check false-positive).

**Fix (pre-patch or stuck case).** Inspect `agent_wakeup_requests` directly (the board used `node+postgres` on port 54329 in QIN-4 to locate the queue state in seconds). Identify the stuck lease; clear it manually; verify the next wake fires.

**Prevention.** Avoid `cancel` whenever possible. When you must cancel, immediately schedule a `monitor` with `monitorNextCheckAt` set 35 minutes out so a wake fires after the lease grace expires. (See `runbook.md` for the exact API payload.)

### 3.4 订阅型接入预算失效 (QIN-3 P1, BOARD-REVIEW-2 §三.3)

**Symptom.** A client integrates via their subscription key (Claude Max, ChatGPT Plus, or a coding-plan endpoint). `costCents` is permanently zero. The budget / circuit-breaker / hard-stop never triggers, even after the agent burns through the client's token quota.

**Root cause.** Custom OpenAI-compatible endpoints have no published price list. Paperclip's billing system computes `billed_cents` from `costCents`; with `costCents=0`, `billed_cents=0`, and the budget enforcement loop is dead code.

**Fix.** Apply a token-to-estimate-dollar conversion at the customer-facing delivery layer (e.g., "1M tokens ≈ $X at retail"). Use that estimate for budget enforcement. Tell the customer in the delivery note that the budget is estimated, not metered.

**Prevention.** When scoping an L3 maintenance contract that uses a subscription key, do not promise metered budget enforcement. Promise estimated budget enforcement with a soft cap. (This is also a contract-text issue, not just a runtime issue.)

---

## 4. Windows-specific traps

Two host-level pitfalls on Windows Git Bash that do not exist on Linux/Mac. Both are well-documented in the production rounds; both will silently corrupt your output if you forget.

### 4.1 UTF-8 corruption in `curl -d` (QIN-4 #10, BOARD-REVIEW-3 §二)

**Phenomenon.** Posting a JSON body that contains any non-ASCII character (Chinese, Japanese, emoji) via `curl -d '...'` on Windows Git Bash: the console encoding corrupts non-ASCII bytes to `?`. The HTTP request "succeeds" but the server receives `?????` in the title or description field.

**Why Windows does this.** Git Bash on Windows pipes through a child shell whose default code page is GBK / cp936, not UTF-8. The `curl -d '...'` argument path goes through that shell and the non-ASCII bytes get transcoded before `curl` sees them. (The `Write` tool, by contrast, writes UTF-8 to disk directly — so file content is unaffected.)

**Solution — temp-file method.** Write the JSON body to a file using the Write tool (UTF-8 safe), then post it:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @body.json \
  "$PAPERCLIP_API_BASE/api/companies/$PAPERCLIP_COMPANY_ID/issues"
```

ASCII-only bodies (IDs, statuses, English text) are safe to keep inline. Bodies containing Chinese (titles, descriptions, comments) must go through the temp file.

### 4.2 `cancel` does not kill detached process trees (QIN-4 #8, BOARD-REVIEW-3 §二)

**Phenomenon.** You cancel a runaway run. The Paperclip run state goes to `cancelled`, but the underlying `opencode` child process and its grandchildren keep running. They may still submit API calls (the QIN-4 case: 4 duplicate hire requests landed before the process died).

**Why Windows does this.** Linux signals the whole process group; Windows requires explicit job objects or `taskkill /T` to walk the tree. The Paperclip adapter's cancel handler does not, by default, create a job object that walks the tree on cancel.

**Solution.** When you must cancel, do it physically:

```bash
# Find the opencode PIDs the run spawned
ps -W | grep opencode
# Kill the tree
taskkill /F /T /PID <pid>
```

Expect to receive 4–5 duplicate API submissions from the dying agent before it actually exits — approve 1, reject the rest. This is normal and was observed in QIN-4 (the agent recovered from the API 400 errors itself, which is a useful resilience signal).

### 4.3 Cold-start timeout (QIN-2, BOARD-REVIEW §四.3)

**Phenomenon.** Every heartbeat spawns a new opencode instance; the first ~15–30 seconds are spent fetching from `models.dev`. Latency spikes on every cold start.

**Mitigation.** Out of scope for the operator skill — this is upstream. Note it in your run-time estimates: budget 30s for cold start before counting actual work time. (QIN-2 still finished in 22 minutes despite this, so it is a budget note, not a blocker.)

---

## 5. Acceptance checklist

When you mark a Paperclip skill package or task `done`, run this checklist first. Each item is falsifiable; if you cannot tick it, do not PATCH `done`.

### For a three-file skill package (SKILL.md + runbook.md + README.md)

- [ ] All three files exist at the path declared in Section C of the brief.
- [ ] `SKILL.md` YAML frontmatter parses cleanly. Required fields: `name`, `description`, optional `license`.
- [ ] `name` in frontmatter equals the directory name (kebab-case).
- [ ] `description` is written in English, contains the Chinese trigger phrases the audience will use, and is bounded by `>` (folded scalar) if it spans multiple lines.
- [ ] `SKILL.md` contains the five blocks: task brief template, organization rules, failure recovery, Windows traps, acceptance checklist.
- [ ] `runbook.md` has at least 8 failure entries, each in a four-column table (症状 / 根因 / 处置 / 预防).
- [ ] `README.md` is bilingual (Chinese + English sections alternating), with a Quick Start section that fits in 5 minutes.
- [ ] Every cited source uses a `QIN-X` identifier or a ref/-relative path. No absolute paths anywhere in the deliverable body.
- [ ] No `Bearer` tokens, no API keys, no customer names, no email addresses, no phone numbers, no real product identifiers appear in any of the three files.
- [ ] No content was fetched from the network during authoring. (Self-check: no `webfetch`, no `websearch`, no external URL hits.)

### For any completed Paperclip task

- [ ] The deliverable file(s) exist on disk at the path declared in Section C of the brief.
- [ ] The deliverable was authored in the same heartbeat as the status PATCH (Rule 2: double-landing).
- [ ] The deliverable contains only content traceable to declared read-only resources (Section B). No fabricated numbers.
- [ ] All issue comments and descriptions posted via `curl` use the temp-file method if non-ASCII (§4.1).
- [ ] Status PATCH response echoed back the new status — if empty response, the write FAILED (§Rule 2 again; see `runbook.md` for the diagnostic).

---

## Appendix — Cross-references to source material

- `QIN-2` (single agent, content creation): BOARD-REVIEW §四 — plan_only misjudgment, think block leakage, cold start timeout
- `QIN-3` (three-agent team, EN expansion): BOARD-REVIEW-2 §三 — handoff gap, subscription budget failure, persona sharing, silent run, think leakage
- `QIN-4` (department-level, AI CS KB): BOARD-REVIEW-3 §二 — unbounded shell, cancel tree kill, lease deadlock, UTF-8 corruption, serial review
- `QIN-11` (self-operating, ops v1.0): BOARD-REVIEW-4 §三 — 10/10 patches verified, zero troubleshooting interventions

See `runbook.md` for the symptom-by-symptom recovery commands.

<!-- v1.1 sanitization patch: F-8 + F-9 by MarketResearcher 2026-09-20 -->
v1.1 sanitization patch: F-8 + F-9 by MarketResearcher 2026-09-20
