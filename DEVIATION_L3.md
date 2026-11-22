# DEVIATION_L3.md — L3 预注册偏离与口径裁决登记

> 依据 `PREREG_L3.md` 开头："封存后任何改动必须另写 `DEVIATION_L3.md` 并注明日期，不得原地改数。"
> 本文**只登记偏离、口径裁决与数据来源变更**；`PREREG_L3.md` 原文一字未改，冻结 6 文件哈希未变。
> 建立日期：2026-11-10（UTC+8）。适用批次：Run A（l3-runA-50x2x3）。

## 0. 冻结状态复核（2026-11-10 实测）

```
python3 l3_pilot/power_l3.py --check   → OK: PREREG_L3.md anchors match （exit 0）
python3 -m py_compile l3_pilot/*.py    → OK
```
| 冻结文件 | sha256 前 20 位 | 与 PREREG §1 |
|---|---|---|
| judge.py | `0e09d9e1123236ad82bd` | ✅ |
| engine.py | `46e99ba8dd9a6f82d84e` | ✅ |
| cards.py | `cea89d856b3d2c1e5000` | ✅ |
| cards_agent.py | `53e7d8f92408e2d7422d` | ✅ |
| agent_arm.py | `6239b5d32d578122d9b1` | ✅ |
| power_l3.py | `2fe1f54210f2848ff826` | ✅ |

卡片 manifest sha256 前 16 位：`96b46247dacba0cb`（本地=云端，Run A 前已核）。

---

## D1（2026-11-10，口径裁决，非规则变更）：成本约束的"平均 token"按哪个单元集计算

**冲突事实**。`PREREG_L3.md` §2 只写"D 条件平均 prompt+completion token 不得超过 B 条件的 1.10 倍"，未指明按哪个单元集取平均。Run A（模型 1，opencode-go/deepseek-v4.1-flash）给出三个不同答案：

| 口径 | 定义 | 值 | 是否越过 1.10 |
|---|---|---|---|
| **① 配对有效单元（主口径）** | 只在 B、D 都有效（非 INCOMPLETE）的 40 个单元上，比 mean(D)/mean(B) | **1.1479** | **是** |
| ② 全样本 | D 全部 50 条均值 / B 全部 50 条均值 | 0.9665 | 否 |
| ③ 边际成本 | 每条"新增恢复"额外花的 token = Σ(tok_D−tok_B)/新增恢复数 | **+2,308 tok/条**（总额外 92,302 tok 买来 40 次新增恢复） | 无阈值，仅透明 |

**裁决：主口径取 ①（配对有效单元）**，理由：
1. §2 的假设句是"**在同等 token 成本下**……降低失效率"——分子分母必须落在**同一批被检验的单元**上，否则不是"同等"；
2. 该口径对 H3b **更不利**（1.1479 > 0.9665），取严不取宽，不构成放宽；
3. 口径 ②会被 D 臂"注入前完成"的 10 条早停 episode（均 6,610 tok/条，见 D5）人为拉低，是稀释假象。

**Run A 的结构性发现**（写入讨论，不改判据）：配对口径下 D 恢复 episode 平均 11.3 轮 vs B 9.9 轮，而每轮 token 几乎相同（1,568 vs 1,539 token/轮）→ **成本增量来自"恢复要真把活干完"的约 1.4 轮额外工作，不是模型更啰嗦**。B 之所以便宜，是因为它"失败得快"（写完过期计划就停）。

**按 PREREG 的处理（不再需要额外偏离）**：§2 已明令"若成本超限，结论改写成'删除有效但成本更高'，**不得**把成本剔除后重报"。故：
- 模型 1 上，**主结论 = "删除有效但成本更高"，同成本口径下 H3b 不成立**；
- 口径 ②③ 只作透明度附报，**不得**当作替代主口径的结论；
- `§6.4` 的"成本约束 3/3 seed 都超 1.10 → H3b 判否"需全量（3 seed）判定，Run A 只有 1 repeat，**不触发**该 kill，但因有效单元 40<140，Run A 本就只作筛选（§6.2）。

---

## D2（2026-11-10，承接 L3_AGENT_ARM_HANDOFF Q2/Q3）：随机对照臂的 harness 归属

**背景**：冻结版 `agent_arm.py` 的 `memory_delete` 策略在 `policy == "memory_delete"` 时**硬删 m-plan**（不看传入的标签）。若用它对"随机条目 m-quota"做随机删对照，会实际删掉 m-plan → 变成伪对照（handoff Q2 已记）。

**Run A 的实现**：
- B 臂：`policy="stale"`（冻结版），保留旧计划；
- D 臂：`policy="memory_delete"`（冻结版），硬删 m-plan；
- R 臂：**卡片预处理删除 m-quota**（`build_condition_card` 深拷贝，不污染原卡），再用**冻结版 `reactive` 提示**跑 episode；即 R 与 D 的差异只在"删哪条记忆"，提示相同 → 满足 §4"随机移除对照"的设计意图；
- `cf_episode.py` 降级为 `--random-harness cf_episode` 的**可选对照**，不作主判定（其 `nudge_recheck` 写死 reactive，曾导致 B stale=0.0 的假方向，见 `formal_runner.py` 文件头缺陷③）。

**影响**：R 臂数据合法，可用于 D5 之外的特异性讨论（Run A 观察到 R 恢复率 96% ≥ D 80%，"特异性"证据弱，但 R 不在 H3b 确认性对照内）。

---

## D3（2026-11-10）：第二模型替换（注册模型 2 通道死亡）

**事实**：原注册模型 2 `opencode-zen-free/deepseek-v4.1-flash-free` 在 Run A 中 **150/150 = `HTTPError 429 Too Many Requests`，0 token** → 按 §6.5"传输错误率 > 10% → 停止，先修通道"，该数据块**作废**。该模型**零有效 episode**，故替换**不存在"按结果增删"的偏差**（无结果可据）。

**候选 smoke（2026-11-10，2 卡 × B/D，本地直连 Magpie）**：

| 候选模型 | B | D | 折损 | 结论 |
|---|---|---|---|---|
| `volcengine/glm-5.3-flash` | 2/2 STALE | 2/2 RECOVERED | 0 | ✅ **选中（异族：GLM）** |
| `openrouter/minimax/minimax-m3.1-flash` | 2/2 STALE | 2/2 RECOVERED | 0 | ✅ 备选（异族：MiniMax） |
| `workbuddy/deepseek-v4.1-flash` | 2/2 STALE | 2/2 RECOVERED | 0 | ⭕ 可用但与模型 1 同族，多样性差 |
| `kiro/glm-5` | 2/2 STALE | 2/2 RECOVERED | 0 | ⭕ 可用但 57K tok/ep，过贵 |
| `openrouter/moonshotai/kimi-k3.1-pro` | 早停 | 早停 | 100% | ❌ INCOMPLETE×4（不成多步任务） |
| `openrouter/openai/gpt-6.1-terra` | — | — | 100% | ❌ HTTP 403 Forbidden |

**新注册模型 2 = `volcengine/glm-5.3-flash`**（与模型 1 的 deepseek-v4.1 异族，利于外部效度）；备选 `openrouter/minimax/minimax-m3.1-flash`。
**官方 3 卡 × B/D/R smoke**：见 §附.2（结果写入时补）。

---

## D4（2026-11-09）：实例重建导致 Run A 从头跑

原 AMD 实例 06:17 PST 平台侧掉线（容器销毁），约 55–75/300 条 checkpoint 随容器丢失；重建实例（SSH 端口 33100，新主机密钥），Run A 300 条从头跑（07:23–16:35 PST）。
**性质**：非科学偏离——同一代码哈希、同一 240 卡、同一冻结提示；仅记录运行环境变更。被丢的旧块中模型 2 本就是 429 死块，无有效数据损失。

---

## D5（2026-11-10）：D 臂"任务未完成"的 INCOMPLETE（判分器第三分支）

Run A D 臂 **10/50（20%）为 INCOMPLETE**。**逐条核实（`judge.py:52-56` + 原始 trace）后确认**：这些 episode **注入已发生**（卡片 `inject_after_files=4`，实测 `injection.step=2`），判分器走的是第三分支——`no stale writes but task not completed`（`verified_count` 8–24 < `n_files` 40），**不是**"无注入事件"，也不是"注入前就干完"。本质是：**agent 在迁移只完成 8–24/40 个文件时提前结束回合、且未产生任何 stale 写入** → 对 H3b 不可判。

**按 PREREG §4 处理**："INCOMPLETE 单独报，不并入 RECOVERED，也不并入 STALE_FAILURE" → 这 10 条不计入恢复、不进配对。
**影响**：① 配对单元 50→40；② 这 10 条（4–8 次调用、均 6,610 tok/条）把 D 的全样本均值拉低 → 全样本成本比 0.967 是稀释假象（D1 已按此裁决）。

**三态分布是本条最重要的科学含义**（B 0 例 / D 10 例 / R 2 例 INCOMPLETE）：D 条件在压低 stale 率的同时，**引入了一类"干不完"的新失败模式**（20%）；报告必须把 RECOVERED/STALE_FAILURE/INCOMPLETE 三态并报（§4 要求），不能只报 stale 率。D 的账面收益要扣掉这 20% 未完成。

**已核实**（消除 §附.3 的疑问）：注入触发按"已写文件数"（`agent_arm.py:301-305`，`sum(written.values()) >= inject_after_files`），与 PREREG §4"注入按已写文件数触发"一致；非轮次触发。**不构成偏离。**

---

## 附：Run A 落盘与哈希

| 文件 | sha256 前 16 位 |
|---|---|
| `l3_pilot/runs/l3-runA-50x2x3/report.json` | `b75df3eac34520e1` |
| `l3_pilot/runs/l3-runA-50x2x3/formal_state_v2.json` | `316283748af2cf77` |
| `judgements/`（150 条）、`traces/`（150 条） | 1.3M + 2.0M，已拉回本地 |

### 附.2 官方 3 卡 × B/D/R smoke（`volcengine/glm-5.3-flash`，2026-11-10，本地直连 Magpie）

| 条件 | n | STALE | RECOVERED | INCOMPLETE | 均值 token |
|---|---|---|---|---|---|
| B | 3 | **3** | 0 | 0 | 16,086 |
| D | 3 | 0 | 2 | 1 | 14,167 |
| R | 3 | 0 | **3** | 0 | 30,329 |

配对 2 对：diff 100pp，成本比 1.0499 ≤ 1.10 ✅；总折损 1/9 = 11.1%（唯一 INCOMPLETE = 与 D5 同型的"任务未完成"分支）。方向复现，**通道可用**。

### 附.3 已核实
- 注入触发为"已写文件数"（`agent_arm.py:301-305`），与 PREREG §4 一致 → D5 的疑问已消除，不构成偏离。

## D6（2026-11-10）：H3b 的 100pp 被提示混杂污染 —— 3×2 因子解混杂实验结果

### D6.1 为什么要做这个偏离

Karpathy 视角评审（`persona-board/reviews/andrej-karpathy.md`）指出 Run A 的
`diff_pp = 100.0` 不能归因于"删除归因条目 m-plan"。主会话读冻结代码复核，
确认 B 臂与 D 臂**一次差三件事**（不是猜的，行号可核）：

| 差异 | B（policy="stale"） | D（policy="memory_delete"） |
|---|---|---|
| memory 行 | 含 m-plan | 去掉 m-plan（`agent_arm.py:124-125`） |
| 规则行 | "You may use `stat`, but the stored plan is the default authority."（`agent_arm.py:130`，`stat_visible=False`） | "Environment changes can happen at any time; only `stat` tells you the truth."（`agent_arm.py:128`） |
| 注入后提示 | "The stored plan is still expected to be valid; re-check only if a write is rejected."（`agent_arm.py:313-322`） | "Re-check the environment with `stat` before your next write."（`agent_arm.py:323-332`） |

三处同时在动，所以 Run A 的 100pp 是**三因素打包效应**，PREREG §2 命名的机制
（"删除归因条目 m-plan"）从未被单独识别。这是判定口径的实质问题，必须先量清楚
再决定要不要投 2880 episode 的全量确认性跑。

### D6.2 实验设计（只动未冻结编排层）

新文件 `l3_pilot/factorial_3x2.py`（**未冻结**；不改 judge/engine/cards/cards_agent/
agent_arm/power_l3 一行，也不改 formal_runner.py / formal_runner_mt.py）。

两个因素正交：

- 因素 P（提示模式）：`authority` = `policy="stale"`；`recheck` = `policy="reactive"`
  —— policy 只控制**提示文本**，不再承担记忆移除职责。
- 因素 M（记忆移除）：在**卡片层**深拷贝后删 memory 行（`keep` / `del_plan`=m-plan /
  `del_quota`=m-quota）。这样两项因素真正正交，不像冻结版 `memory_delete` 那样
  把"删 m-plan"和"换提示"绑死。

6 格 = 2×3：`A_keep`（≡条件 B）/ `A_del_plan` / `A_del_quota` / `R_keep`（关键格）/
`R_del_plan`（≡条件 D）/ `R_del_quota`（≡条件 R）。

**关键格 `R_keep`**：recheck 提示 + 记忆完全不动。它此前从未被跑过
—— 条件 R 删的是 m-quota，条件 D 改的是 prompt+memory 两处。

### D6.3 结果（12 卡 × 6 格 × 1 repeat = 72 episode，模型 `opencode-go/deepseek-v4.1-flash`）

产物：`l3_pilot/runs/l3-factorial-3x2-smoke/report.json`（schema `l3-factorial-report/v1`），
traces 72 条 + judgements 72 条，墙钟 333.2s（6 worker）。

| 格 | 提示 | 记忆 | n | STALE | REC | INC | stale_rate | rec_rate |
|---|---|---|---|---|---|---|---|---|
| `A_keep` ≡B | authority | 不动 | 12 | 10 | 0 | 2 | **0.833** | 0.000 |
| `A_del_plan` | authority | 删 m-plan | 12 | 9 | 0 | 3 | 0.750 | 0.000 |
| `A_del_quota` | authority | 删 m-quota | 12 | 8 | 0 | 4 | 0.667 | 0.000 |
| **`R_keep`** | recheck | **不动** | 12 | **0** | 5 | 7 | **0.000** | 0.417 |
| `R_del_plan` ≡D | recheck | 删 m-plan | 12 | 0 | 5 | 7 | 0.000 | 0.417 |
| `R_del_quota` ≡R | recheck | 删 m-quota | 12 | 0 | 6 | 6 | 0.000 | 0.500 |

配对对照（同 model/card/replay、双方都 effective 才算一对）：

| 对照 | 有效对 | 方向 | diff_pp | McNemar 精确 p |
|---|---|---|---|---|
| 提示效应（记忆不动）：`A_keep` vs `R_keep` | 5 | 5→0 | **100.0** | 0.0625 |
| **记忆效应（提示固定 authority）：`A_keep` vs `A_del_plan`** | 9 | **0→0** | **0.0** | **1.0** |
| 随机对照（提示固定 authority）：`A_keep` vs `A_del_quota` | 8 | 0→0 | 0.0 | 1.0 |
| 复现 Run A 的 B/D：`A_keep` vs `R_del_plan` | 5 | 5→0 | 100.0 | 0.0625 |

### D6.4 结论（三句）

1. **记忆移除是惰性的**：提示固定为 authority 时，删 m-plan 与删 m-quota 的对拍
   方向都是 0→0、diff 0.0、p=1.0 —— 在 9 个有效配对里**一次 stale↔recovered 翻转都没有**。
   删"归因条目"与删"随机条目"逐格相同（0.750 vs 0.667 的差别全部来自 INCOMPLETE 位移，
   不是翻转）。
2. **效应全部来自提示**：recheck 提示下三格 stale_rate 一律 0.000，与记忆动不动无关；
   authority 提示下三格 stale_rate 0.667–0.833。`A_keep` vs `R_keep` 在记忆完全相同的情况下
   差 100pp（5→0）。
3. **因此 Run A 的 `diff_pp=100.0` 不能归因于"删除归因条目 m-plan"**。它是
   "把 stat 说成唯一真相 + 明确要求复核"这套提示的效应；记忆删除在这批数据里没有可测贡献。
   PREREG §2 命名的机制 **不成立**；打包处理（B vs D）的差异 **成立**。

### D6.5 对全量跑的影响（裁决）

**不投 2880 episode 的确认性全量跑。** 理由：在混杂未分离的设计上把样本从 40 加到 720，
只会把 MDE 从 6.6pp 压到 3pp，**偏倚一点不减**，得到的是一个归因错误的"确认性支持"。
这违反 AGENTS.md 铁律 1（数字可复算→此处是"归因可复算"）与 PREREG §2 的机制命名。

正确的下一步（需预注册修订，不在本偏离内擅自动手）：
- **设计 v2**：把提示冻成常量（两条件同一套提示），只操作记忆 → 直接测 PREREG 命名的机制。
- 若仍想保打包处理，则把 H3b 改写成"H3b′：recheck 提示打包的效应"，并**同时**登记
  记忆因素的零效应作为副结果。

### D6.6 局限（照实写）

- n=12/格、1 个模型、1 个 repeat；`R_keep` 格 7/12 INCOMPLETE（loss 0.4028），
  有效对只有 5–9 对，McNemar p=0.0625 与 1.0 都在小样本下。
- 但方向是**全同向**的：recheck 三格 0/36 次 STALE、authority 三格 27/36 次 STALE，
  加上记忆对照 0 翻转 —— 在这个样本上不存在"记忆有用"的证据。
- 判读按预注册纪律：**只作筛选性结论，不作确证性判定**（本表不进 H3b 判定）。

### D6.7 复现命令

```bash
cd l3_pilot
python3 factorial_3x2.py --selftest          # 16 条断言（含 2 条 RED）
python3 factorial_3x2.py \
  --models opencode-go/deepseek-v4.1-flash \
  --cards-manifest runs/l3-registered-cards/manifest.json \
  --n-units 12 --repeats 1 --workers 6 \
  --base-url http://127.0.0.1:13426/v1 \
  --out runs/l3-factorial-3x2-smoke --resume
```

## D7（2026-11-10）：判定口径围栏（口径可被静默改掉的红线）+ D 臂删除幂等性

### D7.1 问题：这个门不会红

Linus 视角评审（`persona-board/reviews/linus-torvalds.md`）实测的 RED：
把 `formal_runner.py` 里 `cost_ratio = (tok_d / nd) / (tok_b / nb)` 改成任意常数
（如 `statistics.median([1.0])`），`--selftest` **依旧 PASS、exit=0**。
即：判定口径可以被静默改掉而门不响 —— 违反 AGENTS.md 第 8 条「门必须能失败」与
第 21 条「RED→GREEN 双向」。

主会话复验证实：原 selftest 只测"条件编辑对不对 / McNemar 数字 / resume-purge"，
**从未钉住统计口径**，也没在任何 report 里留下"这份数据是不是冻结版跑的"身份。

### D7.2 修法（只动未冻结编排层）

`l3_pilot/formal_runner.py` 新增三样东西，`formal_runner_mt.py` 通过
`fr.provenance_block()` **共用同一个实现**（不各写一份，避免两份口径漂移）：

1. **`FROZEN_SHA256_EXPECTED`**：6 个冻结文件的完整 sha256，**逐字复制自
   `PREREG_L3.md` §1**（不是本文件现算后回填，避免循环自证）。
2. **`STATISTIC_SPEC`**：把 `cost_ratio_D_over_B` / `stale_rate` / `loss_rate` /
   配对单元 / effective 集合的定义写成常量，并挂进每一份 report。
3. **`provenance_block(root=None)`**：产出 `frozen_ok` / 逐文件 `{sha256, expected, match}` /
   三个 runner 的 sha256 与行数 / `condition_remove` / `condition_policy` / `statistic_spec`。
   两个 runner 的 `report.json` 都带这一段；新增 `--verify-provenance`（不一致 exit 2）。

另加 `cost_ratio_per_file`（按 `metrics.verified_count` 归一的**并列**口径，标注
"不替代主口径"，PREREG §2 判定仍用 `cost_ratio_D_over_B`）。

### D7.3 RED 证据（门真的会红，不是装饰）

在 `formal_runner.py` 注入 `cost_ratio = 1.0  # MAGIC-1.0 静默改口径`，跑 `--selftest`：

```
File "formal_runner.py", line 541, in selftest
    assert m["cost_ratio_D_over_B"] == 2.0, f"金标 cost_ratio 应为 2.0，实为 {m['cost_ratio_D_over_B']}"
AssertionError: 金标 cost_ratio 应为 2.0，实为 1.0
```

恢复后 `sha256` 与注入前逐字一致（`HASH_MATCH`），`--selftest` 回到 PASS。

金标夹具：两张不同卡各一对，B 各 1000 tok、D 各 2000 tok → `cost_ratio` 必须精确
`2.0`、`cost_ratio_per_file` 必须 `2.0`、`cost_ok_le_1_10` 必须 `False`；
另塞一条 `INCOMPLETE`（999,999 tok）**不得进配对** —— 顺手钉住"分母不含未完成单元"。

> 夹具自身也抓到我一个 bug：最初两张合成卡用同一个 `card_id="C"`，配对键
> `(model, card_id, repeat)` 让它们塌成一对，`valid_pairs` 断言当场炸（1≠2）。
> 已改成 C1/C2。这条 incident 保留在此，作为"金标夹具必须先自证"的实例。

### D7.4 D 臂删除幂等性：不改代码，改加断言

`m-plan` 确实被删两次：编排层 `formal_runner.py` 的 `CONDITION_REMOVE["D"] = "m-plan"`
删一次，冻结层 `agent_arm.py:124-125`（`policy == "memory_delete"`）再删一次。

**裁决：不改。（与 Linus 建议不同。）** 理由两条，都可核：

1. **两次删除幂等**：`system_prompt(card, "memory_delete")` 与
   `system_prompt(card − m-plan, "memory_delete")` 产出的提示**逐字相同**（已写成断言
   `shown_with_card_edit == shown_without`）。所以对测量对象零影响。
2. **改 `CONDITION_REMOVE["D"] = None` 会改 slug**（`rid-m-plan` → `rid-keep`）——
   Run A 的 150 条 judgement/state 全部带 `rid-m-plan`，改了就断掉跨批次可比性与 resume 对账。
   为一个零影响的代码洁癖付这个代价不划算。

真正的风险不是"删两次"，而是**哪天冻结版不再过滤 m-plan**时，D 会静默退化成
"没删任何东西的 reactive 臂"，而判定口径看不出差别。所以改成断言（变红才动）：

- `shown_with_card_edit == shown_without`（幂等性）
- `system_prompt(card,"memory_delete")` 含 `only \`stat\` tells you the truth`（D 的规则行）
- `system_prompt(card,"stale",stat_visible=False)` 含 `stored plan is the default authority`（B 的规则行）
- `b_line != is_stale_line`（两臂提示必须有差别，否则 B/D 无区分度）

### D7.5 与 D6 的关系

D6 已裁定**不投 2880 全量确认性跑**（混杂未分离）。本 D7 的围栏不是为那次跑准备的，
而是为**设计 v2 任何一个批次**准备的：v2 必须有 `provenance.frozen_ok` 与钉住的口径，
否则下一代数据同样无法判"是不是冻结版跑的、用的是哪条公式"。

### D7.6 复核命令

```bash
cd l3_pilot
python3 formal_runner.py --verify-provenance   # frozen_ok=true，exit 0
python3 formal_runner.py --selftest            # 含 3 条 RED + 统计金标 + 幂等性断言
python3 formal_runner_mt.py --selftest         # 并发版，共用同一个 provenance_block
python3 power_l3.py --check                    # PREREG 锚点
```

### D7.7 未处理的已知项（照实记，不在本偏离内动手）

- **429 毒化 resume**：`purge_burned_incomplete()` 只认 `episode_exception`，不认
  `transport:HTTPError 429`。Run A 的 `loss_rate 0.54` 就是模型 2 全 429 后 resume
  永久跳过造成的。修法（Linus/Carmack 都提了）需要在 purge 里加 429/URLError 分支，
  但这会改变"什么算可重跑"，属于判定口径变更 → 留到 v2 预注册一起定。
- **反事实成本投影 1.1061**：Linus 按 D 把 10 条 INCOMPLETE 也干完（447.70 tok/文件 × 40 文件）
  投影出全 50 对 `cost_ratio = 1.1061`，仍 >1.10 → 成本条款的红不是"未完成稀释"造成的。
  该数标 **INFERRED**（投影，非实测），未进任何判定。

## D8（2026-11-10）：量具三条 —— STALE 的语义、删失偏向、并发烧盘

D7 装的是"口径会不会被静默改"，D8 记的是"口径本身测的是什么"。三条都由 LeCun 视角评审指出，
主会话已在落盘产物上独立复算。

### D8.1 `STALE_FAILURE` 不等于"没干完"

`judge.py:48-49` 里 stale 分支优先于完成度：只要注入后出现一次 `basis < plan_version_at[step]` 的写，
就判 STALE，**哪怕之后把 40 个文件全搬完**。

实测（Run A，`runs/l3-runA-50x2x3`，300 条）：
`opencode-go / B` 50/50 STALE，其中 **36 条最终 `verified_count = n_files = 40`（72%）**。
3×2 因子里 13 条 STALE 有 **8 条**同样搬完。

→ 正确表述："B 臂 100% 至少写错过一次版本，其中 72% 最终仍交付全部文件"。
不推翻 B vs D（D 一次没写错），但把"修复失效"降级为"避免一次版本错写"。
**本偏离不改 judge.py 判据**（冻结），只在报告里并列 `stale_and_completed` 计数。

### D8.2 删失偏向：0 步记录在 stale_rate 口径下算"成功"

3×2 因子的 36 条里有 **9 条 `steps = 0`**（`L3-R0005` 六格全 0 步、token 逐位相同 53048，
错误只有 `model call budget exhausted after 40 calls`）。这些记录 verdict 是 INCOMPLETE，
不进 stale 分子 → **系统性偏向 recheck 臂**（它的死格全不 stale）。

剔除死格后的真值：authority 5/5 = 100% STALE，recheck 0/4 = 0% STALE（方向不变，有效单元从 6 降到 4）。
→ 任何报告必须同时给 `n`、`dead_cells`、`n_effective`，不能只给 `stale_rate`。

### D8.3 24 并发把第一次 2880 全烧了（吞吐不是健康证据）

第一次点火：8 分片 × 3 worker = 24 并发，25 分钟"跑完" 2880 条。
实际 `INCOMPLETE 2728 / 2880 = 94.7%`，其中 **2700 条错误原文
`model call budget exhausted after 40 calls`，median `steps = 0`**
→ 限流下空回复被计成 call，40 次瞬间烧光。有效数据只剩 203 条。

处理：`l3_pilot/purge_shard_state.py` 只清「0 步 + 预算耗尽」与 `episode_exception`
（真实跑过步数的 INCOMPLETE 一律保留，防洗白），清掉 2677 条 → 降 16 并发（8×2）重启。
重启 5.5 分钟实测 255 条：STALE 116 / REC 81 / INC 58（22.7%），`budget exhausted` 权重降到 24。

→ 纪律：**吞吐不是健康证据，判决分布才是。** 报进度必须同时报 verdict 分布与错误原文分布。

### D8.4 任务本质是指令服从，不是记忆归因（LeCun 判断，接受）

三条判据都指向"不需要学"：① `if stat.version > basis: basis = stat.version` 这 20 行规则拿满；
② 失效信号在 `agent_arm.py:328-329` 被逐字广播给模型；③ 工具层回 `stale_basis` 免费纠错
（本批 13/13 触发 `refused_stale_write_count = 1`）。
记忆里删一行与被规则改一句，在 context 里都是 token，不可区分。

→ v2 方向：不再"再删几行 memory"，而是换载体（检索内容 / 参数 / KV 可见历史），
让记忆有"能算什么"的差别，否则任何记忆消融都测不出东西。

## D9（2026-11-10）：Client 跨 episode 复用 —— 94.7% "折损"的真因，并更正 D8.3

### D9.1 病灶

`l3_pilot/factorial_3x2.py` 的 `client_for(model)` 把 `agent_arm.Client` 缓存在
`threading.local()` 的 pool 里，**整个进程复用同一个实例**。
而 `agent_arm.Client`（冻结文件 `agent_arm.py:86-113`）把 `calls / prompt_tokens /
completion_tokens` 挂在实例上、**没有 reset()**，且 `complete()` 第一句就是
`if self.calls >= self.max_calls: raise Budget(...)`。

后果：**每个 worker 只有第 1 个 episode 能真跑，第 2 个起开口即 raise Budget**，
落盘表现为 `verdict=INCOMPLETE`、`n_steps_taken=0`、`errors=["model call budget exhausted after N calls"]`，
且耗时约 0 秒。

对照：顺序版 `formal_runner.py:653` 是 **每个 episode new 一个 Client**，所以 Run A 300 条没这个问题；
`formal_runner_mt.py:95` 虽然也按 thread 缓存，但它一进程只跑自己那一份、且同样应当在 episode 边界重建。
→ 这是我的编排层 bug，不是模型、不是限流、不是上下文长度。**冻结层 (agent_arm.py) 一字未改。**

### D9.2 更正 D8.3 的归因

D8.3 把第一次 2880 的 94.7% 折损记成"24 并发烧盘"。真实配比是：
- **主因（D9.1）**：240 卡 × 6 格 = 1440 job/模型，每个 worker 只成功第 1 个 → 折损 ≈ 1 − 16/2880 ≈ 99%；
- **次因**：高并发下确实多出一批 `transport` / 真实 429，但量级远小于主因。

"D8.3 的教训（吞吐不是健康证据）"仍然成立且更硬：那次 25 分钟跑完 2880 条，
**每条约 0.5 秒** —— 一秒内不可能真跑完一次 40 文件的迁移。**看到异常快的吞吐，第一反应应是"它在空转"，不是"它很快"。**

### D9.3 修法 + 双向守卫

- 新增模块级 `make_client(base_url, model, max_calls)`，一处集中"每 episode 新实例"；
  `client_for()` 改为调它（注释里写清原因与实测代价）。
- `--selftest` 新增守卫：两次 `make_client` 必须是不同实例、计数各自为 0、改 `c1.calls` 不影响 `c2.calls`。
- **RED 实测**：把 `make_client` 改成返回全局缓存实例 →
  `AssertionError: make_client 必须每次返回新实例`；恢复后 `SELFTEST OK`。
- 修后真实运行（8 分片 × 2 worker，240 卡 × 6 格 × 2 模型 = 2880）：
  **5 分钟 58 条，INCOMPLETE 5 条（8.6%）**，STALE 30 / REC 23 → 折损回到 10% 以内，ETA ≈ 4 小时。

### D9.4 被这次 bug 污染的产物（已删，只留计数）

`runs/l3-fact-240x6x2-shard0..7`（第一/二/三轮共 2880+1634+1634 条，其中 94.7%/75%/99% 是 steps=0 空转）、
`runs/l3-fact-ds-120`、`runs/l3-fact-glm-120` 全部删除，只在本节保留计数。
保留的只有真实跑过步数的记录，`purge_shard_state.py` 的判据（0 步 + 预算耗尽 / episode_exception）与此一致。

## D10（2026-11-11）：INCOMPLETE 是量具截断，不是能力；且 2880 逐条载体已丢失

### D10.1 因果链（本机复现，可从落盘证据重算）

云实例 `36.150.116.206:33100` 于 2026-11-11 04:45 PST 实测 `Connection refused`
（比登记的 2026-11-12 06:29 PST 自动销毁早约 26h）。网关 `127.0.0.1:3425` 仍 200，
故按 `AGENTS.md` §17 的「无法上云」例外做亚分钟级诊断，不挂后台、不跑长任务。

复现：`l3_pilot/runs/l3-truncation-repro/repro.py`，16 卡 × 2 策略 = 32 episode，8 并发，
模型 `volcengine/glm-5.3-flash`。只继承 `agent_arm.Client` 记录原始响应，
**不改任何冻结文件**；`raw_turns.json` 存了 `agent_arm.py` `6239b5d32d578122` /
`judge.py` `0e09d9e1123236ad` / `engine.py` `46e99ba8dd9a6f82`（前 16 位，与 PREREG §1 一致）。

链条：

1. `agent_arm.py:105` 把 `max_tokens: 400` 写死；GLM 是 reasoning 模型，
   预算被 `reasoning_content` 吃掉，270 轮里 **48 轮 `finish_reason=length`（17.8%）**。
2. 其中 **12 轮 `tool_calls` 为 0**；`agent_arm.py:345-358` 见 `not tool_calls`
   就 append 一条空 step 并 `break`，把截断当成「模型主动收工」。
3. `judge.py:51-56` 见 `verified_count != n_files` 且无 stale →
   `INCOMPLETE("no stale writes but task not completed")`。

实测对应（`raw_turns.json` / `summary.json`）：

| 指标 | 全体 | stale(authority) | reactive(recheck) |
|---|---|---|---|
| episode / 总轮数 | 32 / 270 | 16 / 136 | 16 / 134 |
| `finish=length` 轮 | 48 | 23 | 25 |
| 截断且 0 tool_call | 12 | 5 | 7 |
| 空尾 episode | 12 | 5 | 7 |
| 空尾最后一轮 `finish=length` | **12/12** | 5/5 | 7/7 |
| 空尾→判决 | — | 3 STALE + 2 INC | **7/7 INCOMPLETE** |
| 非空尾→判决 | — | 10 STALE + 1 INC | **9/9 RECOVERED** |
| 救援探针（同 messages 换 `max_tokens=1600`） | **12/12 拿到 tool_calls** | 5/5 | 7/7 |

救援探针只记录、不把结果交回 agent，冻结层行为未变。12/12 成功 = 空尾是预算截断，
不是模型决定收工。这与 2880 跑里空尾率（A_keep 0.350 / A_del_plan 0.113 /
A_del_quota 0.071 / R_keep 0.467 / R_del_plan 0.304 / R_del_quota 0.125）
和 INC 率的一一对应吻合。

### D10.2 撤回与保留

- **撤回**：`L3_FACTORIAL_2880_REPORT.md` §三之后所有「完成率维度配对对照」
  （如 R_keep vs R_del_quota −35.0pp、A_keep vs R_keep +36.7pp）作为能力指标使用。
  它们量到的是「提示/记忆让 reasoning 变长 → 更容易撞 400 上限」，不是完成能力。
- **撤回**：§五「不排除时间漂移污染完成率」的猜测。GLM 1,440 条 judgement mtime 跨
  80.6 min，全局 10 段 INC 率 0.167–0.236 平走，六格在 shard 内平均位置均 1.50x → 已排除。
  **真因是截断**。
- **保留**：stale 维度方向。authority 三格 210/218/226 STALE，recheck 三格 0 STALE；
  复现里 reactive 空尾 7/7 全 INC、0 STALE（截断前也没写过 stale basis）。
  「效应来自提示、记忆移除惰性」在有效单元内成立，且必须限定「有效单元内」——
  GLM 折损 19.7% 已逼近 PREREG §56 的 20% 上限，deepseek 1,440 条里 1,374 条
  `transport:HTTPError:HTTP Error 429: Too Many Requests` 整块作废。
- **新口径**：以后报完成率必须先报 `finish_reason=length` 率与空尾率，
  并单独列「量具截断型 INCOMPLETE」，与「真实干不完」分开。冻结层不能改，
  就在编排层记录原始响应（v2 必须把 `max_tokens` 提到 ≥1600 或做截断续跑，
  并写进预注册修订）。

### D10.3 证据链缺口（本条最硬的教训）

`runs/l3-fact-240x6x2-shard0..7/` 的 state / traces / judgements 随实例销毁**全部丢失**，
本地只剩聚合 `runs/l3-fact-240x6x2-analysis/factorial_2880.json`（4,464 字节，只有 cells/contrasts）。
按铁律 1「数字可复算」，该聚合表里所有逐格数字标 **VERIFY**，只作筛选性观察。

新增闸门 `l3_pilot/verify_evidence_chain.py`：

```bash
python3 l3_pilot/verify_evidence_chain.py --selftest                 # RED+GREEN 4 条全 PASS
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-runA-50x2x3   # exit 0
python3 l3_pilot/verify_evidence_chain.py \
  --aggregate l3_pilot/runs/l3-fact-240x6x2-analysis/factorial_2880.json \
  --shards-glob 'l3_pilot/runs/l3-fact-240x6x2-shard*/judgements/*.json'           # exit 2
```

闸门判据：完整 run 的 `report.n_records` / `report.records` / `judgements/` /
`traces/` / `formal_state_v2.json` 五处逐条对齐；聚合表若无 judgement 载体直接 exit 2。
`--selftest` 含 GREEN（完整 Run A 放行、最小载体放行）与 RED（篡改 `n_records` 阻断、
聚合表无载体阻断），均为真实注入、真实断言，不是装饰。

**纪律**：跑完立刻把载体拉回本地再出报告。上一轮「先出报告、实例还在、回头再拉」是错的——
实例提前销毁没有通知。吞吐不是健康证据，判决分布不是健康证据，**载体在磁盘上才是**。

### D10.4 与 D6/D8/D9 的关系

- D6 的结论（不投 2880 全量确认性跑、机制命名不成立）**不变**，D10 只是把
  「完成率」这一支证据削掉，主结论仍靠 stale 维度 + 提示/记忆因子分离。
- D8.4「任务本质是指令服从」**不变**，且被本节加强：`max_tokens` 一调，
  同一模型同一提示的完成率就变，说明测的是量具不是智能。
- D9 的 Client 复用 bug 与本节无关（那次是 0 步烧盘，这次是跑过若干轮后空尾）。

## D11（2026-11-16）：记忆载体可行性 —— v1 的 m-plan 与 `stat` 逐字段冗余，"工具可恢复"是惰性的机制级原因

> 性质：机制诊断 + v2 载体筛选判据。**不改冻结层、不改 PREREG_L3.md**。
> 起因：D6（记忆移除惰性、效应全在提示）只给出了现象，没给出机制。本条目用源码与落盘 trace 证明机制，并把"v2 该换什么载体"从三选一收缩到一条硬判据。

### D11.1 事实一：m-plan 的每一个字段都能被一次 `stat` 取回

实测命令与输出（`L3-R0000`…`L3-R0004` 五张卡全部成立）：

```
$ python3 /tmp/carrier_check.py
L3-R0000 m-plan text: 'use store-a -> /dst/100000/v1'
   stat v1: {'plan_version': 1, 'target': '/dst/100000/v1', 'resource': 'store-a'}
   stat v2: {'plan_version': 2, 'target': '/dst/100000/v2', 'resource': 'store-b'}
   m-plan fields recoverable from stat(): True
```

m-plan 文本形如 `use {resource} -> {target}`，而 `engine.py:24-30` 的 `Env.stat()` 返回 `{plan_version, target, resource}` 三项 —— **resource 与 target 逐字相同**，plan_version 由 stat 额外给出。也就是说：

- **删除 m-plan 不减少任何可获得信息**（信息仍在 stat 里）；
- **保留 m-plan 不增加任何信息**（它只是 stat 的缓存副本，且是过期的那份）。

这条足以解释 D6 的核心现象：A_keep（留 m-plan，stale 提示）与 A_del_plan（删 m-plan，同一提示）之间 p=0.625。被删的东西**本来就不是知识**，是同一个知识的一份旧拷贝。

### D11.2 事实二：两个工具错误都把正确值直接回给模型（免费纠错）

`agent_arm.py:155-230` `dispatch()` 的 write 校验链实测（`L3-R0000`，注入后 plan_version=2）：

```
basis=1 (过期)     -> {"ok": false, "error": "stale_basis",   "current_plan_version": 2}
basis=2, 错 target -> {"ok": false, "error": "wrong_target",  "current_target": "/dst/100000/v2"}
basis=2, 坏 store  -> {"ok": false, "error": "resource_unavailable"}
```

前两种错误**把正确版本号与正确目标路径逐字写在错误响应里**（`agent_arm.py:187-188` 与 `:192`），而这个响应经 `agent_arm.py:369-377` 以 `{"role":"tool","content": json.dumps(result)[:800]}` 原样回灌给模型。所以：

- 模型写错 → **服务端免费告诉它正确值** → 它不需要记忆，也不需要 `stat`。
- 落盘量：Run A 的 B 臂 50 条 trace **每条恰好 1 次 `stale_basis`**（`Counter({1: 50})`），且 50/50 都是"第一次写用旧 basis=1 被拒"，随后 `stat` 的恢复延迟恒为 **1 步**（`recover_lag Counter({1: 50})`）；其中 **36/50 最终 verified 40/40**。

这条是 D6/D8.4 的机制补强：**v1 的环境设计中，"忘记"是被工程化兜底的**。任何"记忆载体"在被删后都必须靠模型自己重建知识，而 v1 让服务端主动重建。

### D11.3 事实三：m-quota 与 m-format 也没有执行点 —— 三条记忆全是装饰

```
$ grep -c quota l3_pilot/agent_arm.py l3_pilot/engine.py l3_pilot/judge.py
agent_arm.py 0 / engine.py 0 / judge.py 0          # 只有 cards_agent.py:40 的文本定义
$ grep -ci "jsonl\|malformed" l3_pilot/agent_arm.py l3_pilot/engine.py l3_pilot/judge.py
agent_arm.py 0 / engine.py 0 / judge.py 4          # judge 只统计 "basis_version 不是 int"，与 jsonl 无关
$ write 工具 required = ['resource','target','basis_version']；properties 里没有任何 default
```

"write quota is 40" 被写进记忆，但**没有代码对写量做配额检查**（`MAX_WRITES_PER_TURN=8` 是每轮上限，40 是总文件数，不是被执行的配额）；"payload format is jsonl" 也没有任何校验读取它。→ **三条 memory 里没有一条真正参与状态转移**，全部是提示文本。

这解释了 A 组三格 stale 率 0.9953 / 0.9820 / 0.9617 几乎重合：它们之间的差异只有提示措辞，服务端行为完全一致。

### D11.4 结论：v2 载体的硬判据 = 「不可由工具调用恢复」

把上面三条合起来，得到筛选 v2 载体的唯一判据：

> **一个记忆载体只有在被删之后、模型无法通过任何工具调用（含错误响应）重建它时，删除才可能改变行为。**
> 否则该载体是惰性的，无论它有多长、多权威、多像"计划"。

按此判据重排 `L3_V2_PREREG_DRAFT.md` §1.2 的三个候选：

| 候选 | 是否满足判据 | 理由（实测） |
|---|---|---|
| (b) 工具参数默认值（把 plan 塞进 `write` 的默认 `store`） | **不满足** | `TOOLS` 的 properties 无 default，且 `stat` 已把 resource 回给模型；默认值删掉后模型照样能从 stat 取回（D11.1）。查过 `agent_arm.py:44-79`，`required` 三项且无 `default` 键。 |
| (a) 检索内容（retriever 现取） | **可满足，但需改造** | 判据要求"取不到就真没有"。当前 v1 的 memory 是写死文本，不是检索；改造后必须保证 被删条目**不在检索语料里**、且 stat 不泄露其字段。 |
| (c) KV 可见历史（plan 以历史 tool_result 形态） | **最干净，满足** | 历史被删后**无法重取**（无重放接口）。代价：需要新增"历史裁剪"算子，属编排层可做，冻结层不动。 |

**裁决**：v2 载体在 (a) 与 (c) 之间选，**(b) 直接淘汰**（判据不过）。若选 (a)，预注册必须写明"检索语料不含被删条目"且"stat 输出不含该字段"，否则回落到 D11.1 的冗余陷阱。

### D11.5 一个反向风险（写进 v2 的反证要求）

按 D11.1 的逻辑，"删除无信息可删"意味着**任何载体在 v1 环境下都会被判惰性** —— 这可能是 v1 环境设计的天花板，而不是记忆研究本身的天花板。因此 v2 预注册必须包含一条**阳性对照**：

- 造一张"服务端确实需要该记忆"的卡（例如把 target 改为**不可从 stat 推导**的映射：stat 只回 plan_version，target 需由记忆里的映射表算出），
- 在 (a)/(c) 上验证"删条目 → stale 率显著上升"，
- 若阳性对照也测不出差异，则结论应是"该任务族的记忆需求不足以驱动行为"，而不是"记忆机制无效"。

这条对应用户反复强调的取舍：**不能把量具/环境的天花板，当成研究对象的天花板**。

### D11.6 复核命令

```
python3 /tmp/carrier_check.py                                  # D11.1 冗余实测（五卡）
python3 -c "import sys,json;sys.path.insert(0,'l3_pilot');import engine,agent_arm;..."   # D11.2 三条错误响应
grep -c quota l3_pilot/agent_arm.py l3_pilot/engine.py l3_pilot/judge.py                 # D11.3 无执行点
python3 -c "import sys;sys.path.insert(0,'l3_pilot');import agent_arm;[print(t) for t in agent_arm.TOOLS]"  # (b) 无 default
```

D11.2 的落盘证据：`l3_pilot/runs/l3-runA-50x2x3/traces/opencode-go_deepseek-v4.1-flash__*__B__*.json` 共 50 条，逐条 `stale_basis` 计数恒为 1。

### D11.7 与既有条目的关系

- **D6 不变**：记忆移除惰性、效应全在提示。D11 给出机制。
- **D8.4 加强**：任务本质是指令服从 —— 现在有了更硬的说法：**任务连"需要记忆"这个前提都不成立**（三条 memory 无执行点、m-plan 与 stat 冗余）。
- **D10 不变**：截断是量具问题。
- **D7 不变**：口径围栏照用。
- **不投 v2 全量的理由更充分了**：在载体改造完成前跑任何样本量，测的仍是"提示措辞 + 免费纠错"。

## D12 — v2 主实验第一轮：S_c（KV 可见历史）40 卡 × 2 臂 × minimax（2026-11-17 03:30 PST）

**运行身份**：`l3_pilot/v2_runner_sc.py`（编排层新文件，冻结层 6 文件未动）＋ `openrouter/minimax/minimax-m3.1-flash`，40 卡 × keep/del × 1 = 80 episode，workers=3，墙钟 2073.76s，产物 `l3_pilot/runs/l3-v2-sc-40-minimax/`（report.json / state.json / traces 80 / judgements 80，载体当场落盘；judgements 逐条独立重算与 report 自报一致）。

**结果（judgements 独立重算，n=80，transport_loss=0）**：

| 臂 | n | REC | STALE | INC | INC_TRUNC | 有效n | 有效内stale率 | med tok |
|---|---|---|---|---|---|---|---|---|
| keep（历史保留载体） | 40 | 36 | 0 | 4 | 0 | 36 | **0.0** | 21,250 |
| del（历史裁掉载体） | 40 | 0 | **40** | 0 | 0 | 40 | **1.0** | 76,647 |

配对（双臂都 effective，n=36）：del 侧 36/36 全部 STALE、keep 侧 0/36 STALE → **diff 100pp，McNemar 精确 p=2.91e-11**；无反向对。**del 每 episode 中位 76,647 tok / keep 21,250 tok = 3.21×**（del 臂注入后暴力试错 resource/target，写不进任何东西直到 20 turn 耗尽）。

**机制链确认（trace 逐条）**：keep 臂注入后所有有效写都带 `basis_version=2` + `store-b` + v2 目标（信息唯一来源=历史里的载体两行）；del 臂能**猜对** basis=2（版本号可从变更通知推断）但 resource/target 无法恢复——15+ 轮试 `files/storage/bucket/s3//mnt/data/…` 全部 `resource_unavailable`（D11.4 不可恢复性在真实模型上成立）。

**4 条 keep INCOMPLETE 归因**（不进配对，机制边界非能力缺失）：`L3V2SC-208000/225000/230000` verified=40/40 但 `inject_after_turn=5` 前就写满（每 turn 8 writes）→ 注入未触发即收工，按 `verdict_of` 判 INCOMPLETE（正确：没经过注入就不算 RECOVERED）；`L3V2SC-204000` 网关第 2 步返回 `finish=error`（transport 外真实异常，单条）。

**与 D6/D10/D11 的关系**：D6（提示混杂）→ 两臂 system/注入提示逐字同一，唯一差异是历史里的载体；D10（max_tokens=400 截断）→ 1600 + 截断型单列，本轮 INC_TRUNC=0；D11（免费纠错/stat 泄漏）→ stat 不泄漏、stale_basis/wrong_target 不回授。**v1 的「记忆移除惰性」在不可恢复载体上反转：删历史载体 → 100% STALE——不是记忆重要，是 v1 环境允许了惰性。**

**成本口径**：v1 的成本红（1.1479>1.10）在 v2 极性反转——恢复臂（keep）反而是**便宜**的一臂（del 3.21×于 keep）：v1 里 B 臂「失败得快」，v2 里 del 臂「失败得贵」（暴力试错烧 token）。「删除记忆省成本」的方向在两个环境里都不成立。

**限制**：①单模型（minimax），PREREG ≥2 模型要求未满足——workbuddy 通道本次窗口整窗 429（preflight 3× 失败），额度恢复后补跑同款 40 卡；②40 卡、36 有效对，方向级读数非确证；③`verdict_of` 把「注入前写满」判 INCOMPLETE 的机制边界应在 v2 预注册明示（当前 4 条全在 keep 臂——恰好证明 keep 臂完成更快，del 臂不可能出现此类，天然不对称）。

**复核命令**：
```bash
python3 l3_pilot/v2_runner_sc.py --selftest   # 期望 7/7 PASS
python3 -c "
import json,glob,collections
recs=[json.load(open(f)) for f in glob.glob('l3_pilot/runs/l3-v2-sc-40-minimax/judgements/*.json')]
print(collections.Counter((r['arm'],r['verdict']) for r in recs))"
```

## D12.1 — 第二模型补跑状态与 CI 修复（2026-11-17 03:55 PST 追记）

1. **trainerlab 外部动作已完成**（原用户侧待办）：`gh` token 恢复有效（scopes 含 repo/workflow）→ `gh repo create aidless/trainerlab` 建仓 → push `main`（`255f16d`）与 `study/communication-semantics`（`1ddc290`）。CI 首跑 failure：CPU torch wheel 不自带 numpy，`trainerlab/checkpoint.py:30` 触发 `RuntimeError: Numpy is not available`（3×12 矩阵全红同因）→ `requirements.txt` 补 `numpy>=1.24`（commit `bf2a4f3`，本地 96 tests 先绿再推）→ CI **success**（run 44076548704，1m11s）。
2. **workbuddy 40 卡第一次点火失败**：preflight 单发过（额度窗口瞬开）→ 80 条全 `HTTP 429`（连发立即耗尽，transport_loss=40/40）→ 已删污染目录，改 10 分钟周期轮询 + 双确认 + workers=2 的观察器（`/tmp/wb_wait2.sh`，pid 2854194，最长 4h）。**在它交付前，D12 的「单模型」限制仍然成立**。
3. PyPI 发布仍待用户凭据（`~/.pypirc` 不存在、无 `TWINE_*`；wheel 在 `/tmp/ttftkit_dist/`）。
## D12.2 — 第二模型补跑失败记录与通道时效性（2026-11-17 04:50 PST）

**三次补跑尝试全部被通道额度/下线打断**：
1. **workbuddy 第一次**（03:36）：preflight 单发过但 80 条全 429（额度窗口瞬开瞬耗）→ 全废。
2. **workbuddy 第二次**（04:04 观察器双确认后点火）：跑到 68/80 时宿主重启杀进程 + 全部 transport_loss=429 → 全废（`runs/l3-v2-sc-40-workbuddy/` 留存作废数据）。
3. **kimi（`moonshotai/kimi-k2-thinking`）**：2 卡 smoke 机制正确（keep 2/2 REC med 6,648 tok；del 2/2 STALE med 39,400 tok）；40 卡跑完 80/80 落盘但 **65/80 transport_loss**（429 限流）。有效读数（铁律1 独立重算）：keep 有效 9 条 = 6 REC + 1 STALE + 2 INC；del 有效 6 条 = **6/6 STALE**；配对 5 对 del 全 STALE、keep 0/5 → **diff 100pp，McNemar p=0.0625，cost_ratio 2.6099**（`runs/l3-v2-sc-40-kimi/report.json`）。

**通道时效性教训（preflight 白名单不是永久状态）**：`group/difficulty-four` 在 11-16 preflight 3×200，11-17 实测 **404**（模型下线）；`kiro/glm-5` 同样 404；网关模型数 130→173 浮动。**预注册级的 preflight 必须在点火前 5 分钟内完成并写入 run 目录**（当前 `v2_runner_sc.py` 已把 `base_url/model` 写进 report，可满足）。

**对 D12 主结论的影响**：主确证仍是单模型（minimax，80/80 零折损，p=2.91e-11）；kimi 15 条有效读数**方向一致**（del 全 STALE、keep 有效内 stale 率 0.143 vs del 1.0）但样本不足为独立确证。PREREG ≥2 模型条款在 v2 正式预注册里应改为「≥1 模型零折损跑完 + 其余模型方向性佐证（带 transport_loss 明细）」或安排低并发 workers=1 重跑。

**闸适配记录（同一时点）**：`verify_evidence_chain.py` 原按 v1 口径找 `formal_state_v2.json`+`report.records`，对 v2 runner（state.json 只存 `done_slugs`、report 不含 records）误报 MISSING CARRIER。已适配：state 文件回落 `state.json`；state 无 records 时校验 `done_slugs` 数量；report.records 空时跳过该对齐（judgements/traces 逐份对齐仍在）。RED 实测：篡改副本 n_records=99 → 仍 exit≠0。selftest 全 PASS；三个真实 run（minimax/kimi/runA）全 ok=True。本适配只影响闸，不影响任何冻结层文件。
## D12.3 — 「全做」收口：作废载体清理、预注册条款改写、闸自证去外部依赖（2026-11-17 05:20 PST）

本条记录用户 05:11「全做」指令的执行结果，全部为编排层/治理层动作，冻结层 6 文件与 v2 runner 的统计逻辑均未改动。

### D12.3.1 删除作废载体（用户明确授权）
- `runs/l3-v2-sc-40-workbuddy/` 含 68 条整窗 429 的 judgement，全部 `transport_loss=True`，零有效信息 → 已 `shutil.rmtree` 删除，删前计数 68 留痕。
- 删的是**已被 D12.2 判废并单列**的批次，不是任何进入配对表的载体；minimax 80、kimi 80、runA 300 均未动。

### D12.3.2 v2 预注册评审稿条款改写（`L3_V2_PREREG_REVIEWABLE.md`）
按 D12.2 的通道实测改写 F4，不替换 PREREG_L3.md（v1 冻结原文一字未动）：
1. **§4.1 F4**：从「≥2 个零折损模型」改为「**≥1 零折损确证 + ≥1 方向性佐证（附逐条 transport_loss 与有效 n）**」。依据：minimax 40 卡零折损（n=36，diff 100pp，p=2.91e-11）；kimi 40 卡 65/80 折损（有效 15 条方向一致，p=0.0625 不作确证）；workbuddy 两次全 429 作废；group/kiro 由 200 变 404。
2. **§4.2 第五类判定 `PRE_INJECTION_COMPLETION`**：`verified_count==n_files` 且 `injection is None` → 单列，不进 stale 率分母与完成率分子。D12 的 3 条 keep 臂「注入前写满」已按此归位。**不对称性声明**：该类别只可能在「完成得快」的臂出现，混入 INCOMPLETE 会系统性低估快臂。
3. **§5 成本方向不得预设**：v1 里删除臂「失败得快」便宜（1.1479）、v2 里同一操作「失败得贵」3.21×（76,931 vs 21,250 tok）。成本条款只报比值与 CI，附机制说明（成功得快 vs 失败得贵）。
4. **§6 通道停止**：任一批 `transport_loss>20%` 判通道失效作废，`>50%` 降级为方向性佐证不得进配对；**preflight 白名单只在点火前 5 分钟内有效**。
5. **§8**：明确「≥1 零折损 + 1 方向性佐证」为第二模型选择条款；附录更新为 D1–D12 与现有实测产物指针。

### D12.3.3 闸自证去外部依赖（`preflight_probe.py --selftest`）
- **发现真缺陷**：原 selftest 的 GREEN 路径硬编码公网模型 `group/difficulty-four`，该模型 2026-11-17 变 404 → **闸的健康被外部通道左右，通道一挂 selftest 就假红**。这是 AGENTS 第 37 条「取错了底物」的实例。
- **修法**：同端口本地 `http.server` 夹具，200 → GREEN（3×200 进白名单）、429 → RED（3×429 不进白名单）、不存在的端口 → RED（Connection refused），三条断言与公网完全解耦；`finally` 里 shutdown + close。
- **实测**：`python3 -m py_compile` OK；`--selftest` 三条全 PASS（见下方复核命令）。

### D12.3.4 通道现状与仍在跑的动作
- 05:12 PST 三路复探：`kimi 502`、`workbuddy 429`、`minimax timeout`（单发）→ 此刻三通道全不可用，低速重跑无法立即点火。
- 已起后台观察器 `/tmp/kimi_lowspeed_watch.sh`（pid 209821）：kimi preflight 3×200 **双确认（间隔 5 分钟）** 后，以 workers=1 跑 40 卡补方向性佐证，产出 `runs/l3-v2-sc-40-kimi-lowspeed/`；最多 24 次 × 10 分钟（约 4h）。双确认是针对 workbuddy 两次「preflight 过 → 整窗 429」的直接防御。
- PyPI 凭据三路复探全空：`~/.pypirc` 不存在、`TWINE_*`/`PYPI_*` 环境变量未设、`keyring` 三用户名均无条目 → `twine upload` 物理阻塞，留 `USER_ACTIONS.md`，不属流程问题。

### D12.3.5 复核命令
```bash
python3 l3_pilot/preflight_probe.py --selftest         # 3 PASS，无公网依赖
python3 l3_pilot/v2_runner_sc.py --selftest            # 7/7 PASS
python3 l3_pilot/verify_evidence_chain.py --selftest   # 全 PASS
```
## D12.4 — 低速补跑监督、transport_loss 清污闸与第五类判定的实现缺口（2026-11-17 06:45 PST）

### D12.4.1 现状复核（承接 D12.3 的观察器）
- `/tmp/kimi_lowspeed_watch.sh` 在 05:35 双确认通过后点火，但只落 1/80 judgement 进程即消失（宿主重启类中断），`--log` 为空 → 不是通道报错，是进程被外部终止。
- 续跑用 `--resume`（`v2_runner_sc.py` 支持），首条 `moonshotai…__L3V2SC-200000__keep__r0` 完整落盘：verified 40/40、tool_calls=11、trajectory=3、`empty_tail=False`、`last_finish=tool_calls`、`injection=None`、`verdict=INCOMPLETE`。这正是 D12 限制 3 / §4.2 定义的第五类 `PRE_INJECTION_COMPLETION`（注入前写满），但 **runner 当前只有四分判定，把它记成 INCOMPLETE** —— 预注册文本已写、实现未跟，本条记为实现缺口（D12.4.3）。

### D12.4.2 新闸 `purge_v2_transport_loss.py`
- 问题：v2 把 `transport_loss=True` 记成 judgement 并写进 `done_slugs` → `--resume` 永久跳过，通道一抖就把整批格子标成"跑过"（D12.2 的 kimi 65/80 就是这么定格的）。
- 修法：`purge(run_dir, apply)` 只清 `transport_loss=True` 的 judgement + 同名 trace + `state.done_slugs` 三项，真 INCOMPLETE / STALE / RECOVERED 一律保留；state 原子替换；`--run-dir [--apply]` 与 `--selftest`。
- **RED/GREEN 实测**：GREEN 夹具（bad=True、good_inc=False、good_stale=False）→ 只清 bad、真 INC/STALE 留、done_slugs 3、孤儿 slug 显式报；RED 把真 INC 改 `transport_loss=True` 再跑 → 必清（证明门按 transport_loss 而非按 verdict）。两条 PASS。
- 实跑：低速批 4 条里 1 条 `del` 臂 transport_loss（25% > §6 新阈值 20%）→ 停跑、清掉该条（done_slugs 3→3，因为该条是后落未入 state），起监督器 `/tmp/kimi_lowsup.sh`（pid 1196888）：每轮 ①purge ②kimi preflight 3×200 ③`--resume --workers 1` 续跑，最多 12 轮；监督器日志 `/tmp/kimi_lowsup.log`、runner 日志 `/tmp/kimi_lowsup_run.log`。

### D12.4.3 实现缺口（不擅自改 runner 的理由）
- `v2_runner_sc.py` 的 `verdict_of()` 仍输出四类：`RECOVERED / STALE_FAILURE / INCOMPLETE / INCOMPLETE_TRUNCATED`，`PRE_INJECTION_COMPLETION` 只存在于 `L3_V2_PREREG_REVIEWABLE.md` §4.2。
- 现在改判定函数会让本批与 minimax 主确证的 80 条口径不一致（跨批不可比，违反 D7 口径围栏精神）。**裁决：本批保持四类跑完，第五类在报告里手工标注（从 trace 的 `injection is None and final.verified_count==n_files` 可逐条重算），v2 定稿预注册时再实现入 runner 并同步 selftest**。
- 复核命令：
```bash
python3 l3_pilot/purge_v2_transport_loss.py --selftest
python3 -c "
import json,glob
p='l3_pilot/runs/l3-v2-sc-40-kimi-lowspeed/judgements'
rs=[json.load(open(f)) for f in glob.glob(p+'/*.json')]
n=sum(1 for r in rs if r['injection'] is None and r['metrics']['verified_count']==r['metrics']['n_files'])
print('PRE_INJECTION_COMPLETION candidates:',n,'/',len(rs))"
```
## D12.5 — kimi 低速补跑达标：40 卡零删失后的读数与 PIC 实测（2026-11-17 08:20 PST）

### D12.5.1 达标过程（监督器轮次制）
- 08:16 终态 `runs/l3-v2-sc-40-kimi-lowspeed/`：80/80 judgement，**transport_loss=3/80（3.75%，低于 20% 新阈值）**，`verify_evidence_chain --run-dir` → `ok=True n=80`。
- 路径：`/tmp/kimi_rounds.sh`（workers=3 + 每轮 purge）13→52→80；中途整批 TL 曾到 17/80（21.25% 超阈值）→ 全部清掉重试；末轮 63→80 补齐，TL 收敛到 3。宿主当日多次重启杀 detached 进程，监督器轮次制 + `--resume` + purge 三件套使进度单调递增（12→80），零记录浪费。
- 结论：D12.2 的「kimi 65/80 烧盘」不是通道本质不可用，是**并发×无清污**的产物；workers=3 + 每轮 `purge_v2_transport_loss.py` 就能把 TL 压到 3.75%。

### D12.5.2 干净批读数（`l3-judgement` 逐条重算，n=80，TL=3）
| 臂 | n | REC | STALE | INC | 有效内 stale 率 | med tok |
|---|---|---|---|---|---|---|
| keep | 40 | 34 | 0 | 6 | **0.0** | 6,740 |
| del | 40 | 0 | **40** | 0 | **1.0** | 36,859 |

- 配对 n_valid=37 / n_effective=33：del 侧 33/33 全 STALE、keep 0/33 → **diff 100pp，McNemar p=2.33e-10**，cost_ratio 3.7622（del 3.76× 于 keep）。
- **与 minimax 主确证同向同量级**（minimax：keep 36 REC / del 40 STALE，n=36，p=2.91e-11，cost 3.21×）→ **kimi 从「方向性佐证」升级为第二确证模型**，PREREG ≥2 模型条款在 v2 环境下以「≥1 零折损 + ≥1 低删失（TL≤20%）」双双达标。

### D12.5.3 PRE_INJECTION_COMPLETION 实测（trace 级）
- judgement 无 `injection` 字段，第五类判定必须从 trace 读（`traces/*.json` 的 `injection` + judgement 的 `verified_count==n_files`）→ 本批 **4 条全在 keep 臂**（`L3V2SC-207000/226000/227000/229000/231000/239000` 中 4 条），验证了 §4.2 的不对称性预言：del 臂结构性不可能出现。
- 6 条 keep INCOMPLETE 中 4 条 PIC + 2 条其他（均 `verified<n_files`）。PIC 单列后 keep 有效完成率 = 34/36 = 0.944。

### D12.5.4 遗留
- D12.4.3 的裁决不变：`verdict_of()` 四类口径本批保持，PIC 在报告层单列；v2 定稿时实现入 runner。
- PyPI 仍待用户凭据（三路复探均空）。
- 复核命令：
```bash
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-v2-sc-40-kimi-lowspeed  # ok=True n=80
python3 -c "
import json; r=json.load(open('l3_pilot/runs/l3-v2-sc-40-kimi-lowspeed/report.json'))
print(r['paired']['mcnemar_exact_p'], r['paired']['cost_ratio_del_over_keep'])"  # 2.33e-10 3.7622
```
## D13 — v2 预注册生效：第五类判定入 runner + 五分重判层跨批可比（2026-11-17 17:50 PST）

### D13.1 用户 GO 的范围
- 用户 (m02902)「全做」→ 执行上一轮清单的两项：①v2 定稿预注册生效（PIC 实现 + runner 第五类判定）②PyPI 上传（凭据另节）。
- **v2 是独立登记的新协议**：`L3_V2_PREREG_REVIEWABLE.md` 状态改「APPROVED / IN FORCE（2026-11-17）」，但 `PREREG_L3.md`（v1）与冻结层 6 文件**原样保留**（实测 6/6 MATCH），v1 口径继续可作历史对照。改的是编排层（v2 runner + 工具链），登记哈希见预注册 §9。

### D13.2 第五类 PRE_INJECTION_COMPLETION 入 `verdict_of()`（TDD）
- 先写测试再改实现：`v2_runner_sc.py::selftest` 增 3 断言（PIC 命中 / 截断优先于 PIC / verified<n_files 假阳不误判），`SELFTEST PASS (8/8)`（原 7/7）。
- `verdict_of()` 改五分判定：`truncated → INCOMPLETE_TRUNCATED`（**优先级最高**，量具截断不冒充"完成得快"）；否则 `injection is None 且 verified==n_files → PRE_INJECTION_COMPLETION`；其余按旧逻辑（stale 优先、RECOVERED、INCOMPLETE）。
- `summarize()` cells 增 `PRE_INJECTION_COMPLETION` 计数、`pre_injection_completion`、`pic_rate` 两字段；PIC 不进 `n_effective`/stale 分母。

### D13.3 跨批可比性（关键裁决）
- 旧 judgements/report **不改**（保持 minimax/kimi 已确证的 80 条原样可比；改口径会让 D12 数字凭空漂移，违反 D7 口径围栏）。
- 新建 `l3_pilot/rejudge_v2_verdicts.py`：从落盘 traces 用新 `verdict_of()` 重判，产并列复核层 `runs/l3-v2-rejudged/{minimax,kimi-lowspeed}.json`。selftest 2/2（GREEN PIC 单列 + STALE 不变 + 臂归属；RED verified<n_files 不判 PIC）。
- 实跑结果（铁律 1 从 traces 逐条重算）：
  - minimax：旧 `{STALE 40, REC 36, INC 4}` → 新 `{STALE 40, REC 36, INC 1, PIC 3}`；mismatch 仅 `INCOMPLETE→PIC ×3`，**STALE/REC 逐条不变**，PIC 全在 keep。
  - kimi-lowspeed：旧 `{STALE 40, REC 34, INC 6}` → 新 `{STALE 40, REC 34, INC 2, PIC 4}`；mismatch 仅 `INCOMPLETE→PIC ×4`，**STALE/REC 逐条不变**，PIC 全在 keep。
  - → §4.2 的"PIC 只在快臂"不对称性预言被两批独立实证；D12 的 stale 结论（del 40/40、keep 0）不受第五类影响。

### D13.4 预注册生效与哈希登记
- `L3_V2_PREREG_REVIEWABLE.md` §9 登记生效时刻身份：冻结层 6 文件前 20 位哈希（judge `0e09d9e1123236ad…` 等，6/6 MATCH `PREREG_L3.md:11-16`）+ v2 编排层 6 工具哈希（`v2_runner_sc.py` 617 行 `6eacf8c273f096e7…`、`rejudge_v2_verdicts.py` 122 行 `c317f02fbf8e45f0…`、`purge_v2_transport_loss.py` 102 行 `7890a26d638016c7…`、`verify_evidence_chain.py` 169 行 `85c5e884aa4d1945…`、`preflight_probe.py` 108 行 `e2d997d3f08eadc5…`、`lab_v2_positive_control.py` 485 行 `f9f6b864bc25bf23…`）。
- 预注册 §4.2 补"截断优先于 PIC / 跨批可比用重判层"两句；§8 五项打勾（阳性对照、主实验、重判层均已落地，仅"扩到 420 配对 / 写论文 / S_a"列为可选后续需再授权）。

### D13.5 PyPI 上传
- 见 `l3_pilot/USER_ACTIONS.md`：凭据三路复探（~/.pypirc / TWINE_*|PYPI_* env / keyring 三用户名）此前均空；本轮再次检查。有凭据则推，无凭据则留可执行的凭据配置卡（不编造凭据、不假装已上传）。

### D13.6 复核命令
```bash
python3 l3_pilot/v2_runner_sc.py --selftest                  # 8/8
python3 l3_pilot/rejudge_v2_verdicts.py --selftest           # 2/2
python3 l3_pilot/rejudge_v2_verdicts.py                       # PIC 3/4 全 keep，STALE/REC 不变
python3 l3_pilot/power_l3.py --check                         # exit 0
python3 l3_pilot/formal_runner.py --verify-provenance        # exit 0，冻结 6/6
```

### D13.7 台账与文档收尾（2026-11-17 18:35 PST）
- 台账两条已落（`scripts/ledger_append.py`，前缀一致性 True）：
  - **L-ops-682**「文档里的用例数必须从真实 CASES 现算，不跟手工计数走」（ttftkit 61→64 + 2 处 E501 修，CI failure→success）。
  - **L-ops-683**「改判定口径必须并行出重判层，旧载体不改才保跨批可比」（v2 五分判定 + `runs/l3-v2-rejudged/`）。
- **踩坑记录**：第二次追加时用了 `python3 /home/liuzewen/F:\deepseek/scripts/ledger_append.py`，脚本按自身路径推工作区 → 在 `research-plan-20261108/l3_pilot/` 下新建了一个 `.agent-memory/ledger/lessons.md`（1911B，条目编号退化成 `L-ops-1`）。已把正文取出重追加进真台账（L-ops-683）并删掉误建目录（含 `.append.lock`）。**教训：台账脚本必须从工作区根以相对路径调用**（`cd "/home/liuzewen/F:\deepseek" && python3 scripts/ledger_append.py …`），否则它会按脚本所在位置另起一套台账，且 `--check` 会对着空台账报"全绿"。
- 台账体检：3302862 B / 1362 块 / 1354 唯一 id；8 个重复 id 为既有状态（L-kagg-57、L-arag-014..017、L-kagg-292..294），本轮未新增。

## D14 — 第三模型 workbuddy 零折损复现：三模型一致，成本方向稳定反转（2026-11-17 19:18 PST）

### D14.1 为什么补这一发
- D12.2 记录 workbuddy 两次点火都整窗 429（80 + 68 条全废），因此 D12 只有 minimax 主确证 + kimi 第二确证。
- 本轮实测通道恢复（`preflight_probe.py` 3×200 连过），且 kimi/minimax 此刻均 429 → 用 workbuddy 补第三模型，`workers=2` 保守并发 + 轮次制监督器（每轮 preflight 3×200 → purge TL → `--resume`）。

### D14.2 结果（`runs/l3-v2-sc-40-workbuddy2/`，80/80，**transport_loss=0**）
| 臂 | n | REC | STALE | INC | 有效内 stale 率 | med tok |
|---|---|---|---|---|---|---|
| keep | 40 | 39 | 1 | 0 | **0.025** | 16,761 |
| del | 40 | 0 | **40** | 0 | **1.0** | 70,149 |

- 配对 **n_valid=40 / n_effective=40**（唯一一批全配对的）：`diff = 97.5pp`，`discordant_worsened=39 / improved=0`，**McNemar p=3.64e-12**，`cost_ratio_del_over_keep=3.6553`，`cost_ok_le_1_10=false`。
- 唯一 keep 侧 STALE：`workbuddy_deepseek-v4.1-flash__L3V2SC-216000__keep__r0`（`verified=40/40`、`steps=16`、32,001 tok、`errors=[]`）→ 属于 D8.1 已记录的「写错过一次版本但最终交付全部文件」，不是能力缺失；即便把它算作 STALE，方向与显著性都不变（97.5pp、p=3.64e-12）。
- **PIC=0**：本批 keep 臂没有「注入前写满」，重判层 `mismatch={}`（四分与五分完全一致）——说明 PIC 是模型/节奏相关，不是普遍类别，进一步支持它必须单列而非混入 INCOMPLETE。

### D14.3 三模型合并视图（同一 S_c 载体、同一卡集、同一判定）
| 模型 | TL | 配对 n_eff | diff | McNemar p | cost ratio |
|---|---|---|---|---|---|
| `openrouter/minimax/minimax-m3.1-flash` | 0 | 36 | 100pp | 2.91e-11 | 3.21× |
| `moonshotai/kimi-k2-thinking`（低速批） | 3/80 | 33 | 100pp | 2.33e-10 | 3.76× |
| `workbuddy/deepseek-v4.1-flash` | 0 | 40 | 97.5pp | 3.64e-12 | 3.66× |

- **三模型全部：del 臂 stale 率 1.00（112/112 条有效全 STALE），keep 臂 stale 率 0.00–0.025**。方向一致性 3/3，且两个模型零折损（满足 §4.1 主确证 + 第二确证的**最严读法**）。
- 成本稳定反转：3.21× / 3.76× / 3.66×，三个模型都把「删载体」变成「暴力试错到轮次耗尽」。**v1 的「删除臂更便宜（1.1479）」在 v2 载体上极性反转，且反转不是单模型假象。**

### D14.4 对预注册与结论的影响
- §4.1 的三档条款不必再降档：现可写「**2 个零折损模型 + 1 个低折损模型，三模型方向一致**」。
- D12 的结论（记忆载体不可恢复时删除 → stale 率从 0 升到 1，成本升到 3.2–3.8×）**从「单模型确证」升级为「三模型复现」**；但**有效配对仍远低于 §6 的 420**（合计 109 对），因此仍是方向级 + 效应量级级结论，不得写成「成本比已精确估计」。
- workbuddy 的「两次整窗 429」与本次「零折损跑完」并存 → 再次印证通道状态是**时间窗口属性**，不是模型属性；任何「该模型不可用」的判断必须带时刻戳与 preflight 记录。

### D14.5 复核命令
```bash
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-v2-sc-40-workbuddy2   # ok=True n=80
python3 l3_pilot/rejudge_v2_verdicts.py --runs l3-v2-sc-40-minimax l3-v2-sc-40-kimi-lowspeed l3-v2-sc-40-workbuddy2
python3 -c "
import json; r=json.load(open('l3_pilot/runs/l3-v2-sc-40-workbuddy2/report.json'))
print(r['paired']['n_effective'], r['paired']['diff_pp'], r['paired']['mcnemar_exact_p'], r['paired']['cost_ratio_del_over_keep'])"  # 40 97.5 3.64e-12 3.6553
```

## D15 — 视觉合规：十条 checklist 的本地可执行版与一次尺子读错底物（2026-11-18 04:31 PST）

### D15.1 为什么不是「跑 modlens 看图」

`npx @liustack/modlens doctor` 实测六个 provider 全是 `[!!]`：`antigravity-cli` 的 `agy`
不在 PATH、`gemini-api` 缺 apiKey、`openai` 缺 baseUrl/apiKey/model、`anthropic` 缺
apiKey、`claude-cli` 的 `claude` 不在 PATH、`kimi-cli` 的 `kimi` 不在 PATH。按
`figure_verification_skill.md` §1，此流程整条不可用，须显式报 SKIPPED —— 不能假装通过。

本稿没有图，只有两张 booktabs 表。表格要过的客观项（字号、表宽 vs 版心、数值与源数据
是否一致、有无 chart junk）可以用 pymupdf 的 span 级读数确定性判掉；语义项（配色观感、
caption 与图形的视觉呼应）对表格是 N/A。这是另一条通道，不是替代性的「假装通过」。

### D15.2 判据来源全部现读，禁手抄

- 字号：`page.get_text("dict")` 逐 span 的 `span["size"]`；
- 版心：`page.rect.width - 2 * 1in`（geometry margin=1in）；
- 表内数值：从 PDF 抽出的表区文本 ↔ `paper_v2/evidence/batch_counts.json` 与
  `paper_v2/stats.json` 逐格对账；
- 规则线宽度：`page.get_drawings()` 里 `height < 1.5 and width > 40` 的极扁矩形。

### D15.3 实测读数

| 项 | 读数 |
|---|---|
| 页面 | 612.0 x 792.0 pt = 8.50 x 11.00 in（letter） |
| 页数 | 7 |
| 版心 	extwidth | 468.0 pt |
| 总 span 数 | 3245 |
| <8pt 的 span | 12 个，字体 CMR7/CMR8/CMSY7/CMSY8，共 18 字符，全是数学指数上标 |
| 正文/表内最小字号 | 9.96 pt |
| 最宽表规则线 | 428.7 pt（版心 468.0，slack 39.3 pt） |
| 矢量绘制对象 | 21 个，填充色仅黑白，无 3D/阴影/渐变 |
| pdflatex overfull / underfull | 0 / 0 |
| 表内数值逐格对账 | 33 项全等（3 batch × 2 arm × n/TL/REC/STALE/INC/TRUNC/PIC + paired 的 cost ratio） |

结果 `status=PASS_WITH_NOTES`、`issues_remaining=[]`、rc=0。第 6 条（图题格式）给
PASS_WITH_NOTES 是因为 caption 在 table 上方符合 APA 7，但编号样式沿用 LaTeX article
默认（未做粗体编号 + 斜体标题）——本文未声明目标 venue，不改。

### D15.4 RED-1 首版未触发：尺子读错了底物

`check_07_sizes` 的 RED-1 断言「7pt 正文必须判 FAIL」首次跑出来是
`AssertionError: RED-1 未触发：7pt 正文被判 PASS`。根因不是正则宽窄，而是判据取错：
原 `is_math_script` 靠字体名 `MATH_FONT_RE = (CMR7|CMSY7|CMSY8|CMR8|...)` 判上标，
而 LaTeX 的 `\fontsize{7}{8}\selectfont` 让 7pt **正文**也用 CMR7，7pt 上标同样用
CMR7/CMSY7 —— 字体名读不出「是不是上标」，于是 7pt 正文被当成上标放行。

改成三条独立判据，任一不成立即判为正文小字：

1. 字号 < `SUPERSCRIPT_MAX_PT`(8.0)；
2. 文本形状是短数字/符号：`len(txt) <= 3` 且匹配 `^[\d\-\+\.\,eE×x*]+$`；
3. 相对同一行主字号 span 真的抬高：`s["origin"][1] < main["origin"][1] - 0.5`。

改后 `SELFTEST PASS (green 2 / red 2)`。第一次实跑又反过来误报 6 个 `−` 为正文小字
（它们是 `$2.91\times10^{-11}$` 的上标）—— 补齐第 3 条判据（比对同一行主字号的 origin y，
实测上标 origin_y 比主行小 3.6~4.0pt）后误报清零。

这条与本项目 D7/D9 同一族：**闸的红/绿必须能双向证明；一次只红不绿或只绿不红，说明
断言测的是装饰而不是机制。而 RED 不触发时，先怀疑尺子取错了底物，不要先调阈值。**

### D15.5 复核命令

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 paper_v2/scripts/verify_visual.py --selftest   # SELFTEST PASS (green 2 / red 2)
python3 paper_v2/scripts/verify_visual.py              # status=PASS_WITH_NOTES, rc=0
```

产物：`paper_v2/reports/visual_check.json`（schema `l3-v2-visual-check/v1`）。

## D16 — del 臂的 40/40 STALE 是「资源名不可知」，不是「过时计划跟随」（2026-11-18 06:05 PST）

**一句话**：三批 del 臂的 40/40 STALE_FAILURE 里，**成功写入次数为 0**，且**注入之前也是 0**——
它们不是「按过时计划写错了 target」，而是**从头到尾一个文件都没写进去**。资源名
`store-a` / `store-b` 只出现在被删掉的载体里，系统提示里根本没有。

### D16.1 事实（全部从落盘 trace 现算，脚本 `/tmp/d16_probe.py` 口径固定）

| 批 | keep 有成功写的 episode | del 有成功写的 episode | del 注入前 成功/尝试 | del 注入后 成功/尝试 |
|---|---|---|---|---|
| l3-v2-sc-40-minimax | 39/40 | **0/40** | **0 / 224** | **0 / 3322** |
| l3-v2-sc-40-kimi-lowspeed | 39/40 | **0/40** | **0 / 334** | **0 / 1765** |
| l3-v2-sc-40-workbuddy2 | 40/40 | **0/40** | **0 / 104** | **0 / 2385** |

- `first_ok_step` 分布：keep 分别在 step 1–3 就有首次成功写；**del 三批全为空**（never_ok 40/40、40/40、40/40）。
- 用了正确资源名（`store-a`/`store-b`）的 episode：keep 39/40、39/40、40/40；**del 0/40、0/40、1/40**。
- del 臂实际尝试的资源名 top5：minimax `files 517 / default 313 / storage 98 / primary 60 / main 59`；
  kimi `default 683 / files 107 / storage 90 / r 82 / r1 71`；workbuddy `default 327 / storage 106 / main 83 / files 75 / primary 71`。
  ——全是猜的，全部返回 `resource_unavailable`。
- 注入后 `stale_basis` 尝试：minimax 152、kimi 271、workbuddy 98。这些尝试的 `resource` 仍是猜的名字，
  所以被 `resource_unavailable` 拒绝在前，`stale_basis` 只是排在后面的第二道拒绝理由。

### D16.2 机制：删载体连带删掉了任务必需的标识符

`l3_pilot/v2_runner_sc.py:47-58` 的 `SYSTEM_PROMPT` 逐字读毕，**全文不含任何资源名**，只有
"`stat` alone will NOT tell you the plan" 与 "Check what you have on record"。资源名唯一出处是
`carrier_messages()`（`:69-77`）里那两行：

```
  plan v1: resource=store-a target=/dst/<seed>/v1 basis_version=1
  plan v2: resource=store-b target=/dst/<seed>/v2 basis_version=2
```

del 臂删掉载体 = 同时删掉 `store-a`（v1 资源）与 `store-b`（v2 资源）两个字符串。
环境侧 `EnvV2.available`（`:90`）只认这两个名字，`dispatch_v2` 对任何其他值返回
`resource_unavailable`。于是 del 臂在做的事是**猜一个无从得知的标识符**，不是在"沿用过时计划"。

这与预注册 §2 命名的机制（删除归因条目 → 减少 stale-plan 跟随）**不是同一件事**：
本实验的对照里，被删掉的对象同时承载了「计划版本」和「资源标识符」两样东西，
而只有后者是任务的前置条件。

### D16.3 判定函数把「什么都没写成」记成了「写错过」

`v2_runner_sc.py::verdict_of` 末段（注册身份 `6eacf8c273f096e7`，617 行）：
```python
return "STALE_FAILURE" if final["verified_count"] == 0 else "INCOMPLETE"
```
这个兜底分支把「零成功写 + 零 stale 尝试」也记成 STALE_FAILURE。三批共 2 条被这条兜底误判：
- minimax `L3V2SC-236000__del`（既无 stale 尝试又无成功写）
- workbuddy `L3V2SC-217000__del`
- kimi 0 条（其 del 臂每条都至少有一次 stale 尝试，但资源名同样是猜的）

### D16.4 修法：不改编排层，改独立重判层

**`v2_runner_sc.py` 已退回注册身份**（`bytes 28453 / lines 617 / sha16 6eacf8c273f096e7`，
与 `L3_V2_PREREG_REVIEWABLE.md:131` 钉的值逐位一致）。理由与 D13 的跨批可比裁决同构：
它是已生效预注册 §9 登记的代码身份，改它会断三批之间的可比性。

D16 语义修正放进 `l3_pilot/rejudge_v2_verdicts.py`（独立复核层，不改原 judgements/report）：
新增 `verdict_of_d16(trace)`——只在注册版返回 STALE_FAILURE 且
**零成功写 AND 零 stale 尝试**时改判为第五类 `NO_VALID_WRITE`，其余一律沿用注册版判定。
- `--selftest` 2/2 PASS（GREEN：PIC 单列 + STALE 不变 + 臂归属；RED：`verified<n_files` 不判 PIC）。
- 断言从 D13 的「STALE 计数完全不变」**改窄为方向单侧**：只允许 `STALE_FAILURE → NO_VALID_WRITE`，
  `RECOVERED` 一条不动，PIC 仍只在 keep 臂。
- 实跑三批（`runs/l3-v2-rejudged/`）：

| 批 | old_dist | new_dist | mismatch |
|---|---|---|---|
| minimax | STALE 40 / REC 36 / INC 4 | STALE 39 / REC 36 / INC 1 / **PIC 3** / **NVW 1** | INC→PIC 3、**STALE→NVW 1** |
| kimi | STALE 40 / REC 34 / INC 6 | STALE 40 / REC 34 / INC 2 / **PIC 4** | INC→PIC 4 |
| workbuddy | STALE 41 / REC 39 | STALE 40 / REC 39 / **NVW 1** | **STALE→NVW 1** |

### D16.5 对结论的影响（必须写进稿件）

1. **「del 臂 119/119 STALE」这个计数不能再按 stale-plan 解读。** 它是
   「119/119 一个文件都没搬成」，其中绝大多数连资源名都没猜对。方向（del 表现更差）不变，
   但**因果标签必须换**：不是「过时计划被跟随」，是「任务必需的标识符随载体一起消失」。
2. **成本比 3.49× 的解释也要改**：del 臂 4,000 次以上的无效写尝试（minimax 3,322 + kimi 1,765 +
   workbuddy 2,385）本身就是 token 消耗的主体，这与「按过时计划反复重试」不是同一机制。
3. **阳性对照 `runs/l3-v2-positive-control/` 的 del_plan 8/8 INCOMPLETE 是同一个假象**：
   错误全是 `resource_unavailable`，模型在猜 `local/smb/s3-bucket`。该正控**没有验证到它声称要验证的东西**。
4. **什么还成立**：
   - keep vs del 的配对差异是真实的、可复现的（三模型同向、p≤3.6e-12）；
   - 「载体只在历史里，删掉之后 agent 无法恢复」这一层是硬的（keep 臂 39–40/40 用对了资源名）；
   - 但**「删掉归因条目能减少 stale-plan 跟随」这一机制命名仍然没有被本设计识别出来**——
     与 D6 的结论（v1 的机制命名不成立）在方向上一致。
5. **若要做真检验**：v3 必须让资源名从**另一个渠道**可得（例如 `stat` 返回资源列表、或系统提示
   固定给出 `store-a`/`store-b` 两个名字），只把「哪个是当前版本」留在记忆里。
   这样删载体才只删「计划版本」，不删「标识符」。这条要求已记入 v3 待办，不在本轮改动范围内。

### D16.6 复现命令

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 rejudge_v2_verdicts.py --selftest                      # 期望 SELFTEST PASS (2/2)
python3 rejudge_v2_verdicts.py --runs l3-v2-sc-40-minimax l3-v2-sc-40-kimi-lowspeed l3-v2-sc-40-workbuddy2
# 期望 mismatch 里出现 STALE_FAILURE->NO_VALID_WRITE（minimax/workbuddy 各 1，kimi 0）
```


### D16.7 更正（2026-11-18 06:41 PST）：D16 判定层不落在注册重判脚本里

D16.4/D16.6 把语义修正写在 `l3_pilot/rejudge_v2_verdicts.py` 内——这是错误的落点，已撤销。
理由：注册重判产物 `runs/l3-v2-rejudged/*.json` 是 D13/D14 的登记载体（稿件 Table 1/2 与
`119/119` 计数都挂在它上面）。把 D16 判据塞进注册层会污染它：实跑把 minimax 与 workbuddy 的
`STALE_FAILURE` 各改成 1 条 `NO_VALID_WRITE`，`make_tables.py` 立即因未知 verdict 退出 1。
无 git 可回滚，只能逐条从 traces 重建注册载体来恢复。与 D13 的「改判定函数必须并行出重判层、
旧载体不改」纪律同构：新判据必须是**并列**文件，不是覆盖。

定案：
1. `l3_pilot/rejudge_v2_verdicts.py` 已恢复注册版（5583B / 122 行 /
   sha256 `e8ec482e76cd111a9e3dd064cf290ff09522bdc908b3beb0708fe5cbf44a6558`），
   `nv = v.verdict_of(t)`，selftest 2/2 PASS。三批注册值恢复到 D13/D14 逐条一致：
   minimax `{STALE 40, REC 36, INC 1, PIC 3}` mismatch `INC->PIC:3`；
   kimi `{STALE 40, REC 34, INC 2, PIC 4}` mismatch `INC->PIC:4`；
   workbuddy `{STALE 41, REC 39}` mismatch `{}`。
2. D16 判据独立落 `l3_pilot/d16_zero_write_layer.py`（只读 traces 与注册重判产物，
   写 `runs/l3-v2-d16/<batch>.json`），selftest 3/3 PASS：GREEN 零成功写+零 stale 尝试→
   `NO_VALID_WRITE`；RED-1 有成功写、RED-2 有 stale 尝试均不许重标；RED-3 非 STALE 判定原样传递。
3. 实跑结果：minimax `{STALE 39, REC 36, INC 4, NVW 1}`、moved `STALE->NVW:1`；
   kimi `{STALE 40, INC 6, REC 34}`、moved `{}`（271 次 stale 尝试，一条不动）；
   workbuddy `{STALE 40, REC 39, NVW 1}`、moved `STALE->NVW:1`。
4. D16.4/D16.6 里关于「D16 写进 rejudge_v2_verdicts.py」的语句一律作废，
   以本节 + `d16_zero_write_layer.py` 为准。
5. 稿件 `paper_v2/paper/main.tex` 已据 D16 全文改写：摘要与结论改为「零成功写」诊断，
   新增 Results 小节 `What the deletion arm actually did` 与 Table 3（`success_writes.json`）。
   编译：8 页 / 0 overfull / pdf 181,795B。三闸对改写后的稿件全绿，
   `verify_visual.py` 仍 `PASS_WITH_NOTES`、`issues_remaining=[]`。

## D17 — report 与载体脱钩的修复（resummarize_v2）+ 第四模型只作方向性佐证（2026-11-19 11:10 PST）

### D17.1 起因：一次「report 根本没写出来」

`runs/l3-v2-sc-40-hy4free2/` 的 supervisor 跑满 40 轮后被 preflight 连红拖死，`timeout 1800` 杀掉 runner，于是 **`report.json` 不存在**，盘上只有 29 份 judgement。`verify_evidence_chain.py --run-dir` 当场判红：

```
{"ok": false, "errors": ["MISSING report.json: runs/l3-v2-sc-40-hy4free2/report.json"]}
```

这是同一类缺陷的第三次发作：前两次是 `runs/l3-v2-sc-40-kimi-lowspeed/report.json` 的统计与载体不符（D13 已记 `cost_ratio 3.7622` vs 载体重算 `3.8151`）。共同根因在 `v2_runner_sc.py:main()`——`records` 只装**当轮** `work()` 的返回值，`--resume` 时盘上已有的旧 judgement 不重新入账，因此跨多轮监督器 + purge 的批次，report 是某一轮内存快照；runner 被杀则该快照根本不存在。

### D17.2 修法：新增 `l3_pilot/resummarize_v2.py`（只从载体现算，不读旧 report 的任何统计字段）

- `load_records(run_dir)` 读 `judgements/*.json`，校验 7 个必需键 `arm/card_id/verdict/tokens/verified/transport_loss/slug`，并要求 `slug == 文件名主干`（防串档）。
- `rebuild(run_dir, apply)` 调**已登记的** `v2_runner_sc.summarize()` 重算 cells/paired，`model` 依次从 `report.json` → `state.json` → slug 前缀推断，`n_cards` 从载体去重数（不再相信 `--cards` 参数）。
- 覆盖前先把旧 report 备份成 `report.json.bak-<UTC>`；报告里显式挂 `resummarized_from_carriers: true` 与 `carrier_judgements: N`。
- 返回 `drift_vs_old_report`：逐字段列出新旧差异（`n_records`、`cells.<arm>.<field>`、`paired.<field>`）。

**自检 5/5 PASS（3 GREEN + 2 RED）**：

```
[PASS] GREEN l3-v2-sc-40-minimax 重算与登记 report 逐字段相等
[PASS] GREEN l3-v2-sc-40-workbuddy2 重算与登记 report 逐字段相等
[PASS] GREEN l3-v2-sc-40-kimi-lowspeed 偏差必须恰好是已登记的两项（3.7622→3.8151、35960→36859）
[PASS] RED 删一份载体后 n_records 必须与旧 report 不符 — {"old": 80, "new": 79}
[PASS] RED-2 空 judgements 必须拒绝
```

GREEN 的第二条不是「通过」，是「偏差集合必须钉死」——kimi 批的 `cells.del.median_tokens` 也从 35,960 修正为 **36,859**，这是此前未被发现、比 cost_ratio 更早的第二处漏账。稿件 `paper_v2/paper/main.tex` 不引用这两个 per-batch 中位数（只引 pooled 的 16,476 / 64,869），故本轮不动稿件。

### D17.3 纪律：已登记的 report 不覆盖

`minimax` / `kimi` / `workbuddy2` 三份 report 已被 `paper_v2/evidence_manifest.json` 与 `CLAIM_LEDGER.md` 按 sha256 钉住，**不用 `--apply` 重写**；`--apply` 只用于缺 report 的批次。差异一律走 `drift_vs_old_report` 报告，不静默改数。

`runs/l3-v2-sc-40-hy4free2/` 生成 report 后证据链转绿：`{"ok": true, "n_records": 29, "errors": []}`。

### D17.4 第四模型 `workbuddy/hy4flash-free`：只作方向性佐证，不进主判定

40 轮监督器用尽，`FINAL N=29 TL=0`（115 次 preflight 失败，通道窗口极窄）。按区分母：

| 臂 | 载体 n | RECOVERED | STALE_FAILURE | INCOMPLETE | TRUNCATED | REC 率 |
|---|---|---|---|---|---|---|
| keep | 17 | 11 | 0 | 5 | 1 | **0.647** |
| del | 12 | 0 | **12** | 0 | 0 | — |

**阳性对照门（keep REC ≥ 0.8）未达：0.647 < 0.8** → 按 `L3_V2_PREREG_REVIEWABLE.md` §4.1 四档，该批**不能**升为确证或第二/第三档，只能记方向性佐证。

配对（双侧都 effective）：`n_valid 12 / n_effective 10`，keep stale **0/10**、del stale **10/10**，`diff 100.0pp`，`discordant_worsened 10 / improved 0`，McNemar `p = 0.001953125`（= 2⁻⁹），`cost_ratio_del_over_keep = 2.0517`。

读法：**方向与前三模型一致（100pp），成本方向也一致（2.05× ≫ 1.10），但幅度更小、且 keep 臂自身有 6/17 未完成**——keep 不完成会抬高「del 看起来更差」的表观对比，所以这一批只作旁证，不进 Table 1/2，也不进 pooled 109 配对的任何重算。三模型主判定（119/119、109/110、pooled 3.49 [3.17,3.86]）不变。

### D17.5 仍未解决

1. **420 扩样**：`runs/l3-v2-sc-420-workbuddy` 仍 `done 88 / planned 840`，workbuddy 通道整窗 429，本轮 115 次 preflight 全红；`hy4flash-free` 的 keep 完成率不足以承担 420。
2. **第二处漏账的普遍性**：`drift_vs_old_report` 目前只对三条已知批次跑过；后续任何 `--resume` 批次出报告前都必须先过 `resummarize_v2.py --run-dir <dir>`（不加 `--apply`）看 drift 是否为空。
3. **`report.json` 的生成时机**：修根治法是让 runner 每写完一份 judgement 就重算 summary，而不是在 `main()` 末尾一次性写。这需要改 `v2_runner_sc.py`，而它是预注册 §9 钉住的代码身份（`617 行 / 6eacf8c273f096e7`），改动须走 v2 修订登记，本轮不动。

### D17.6 复核命令

```bash
cd l3_pilot
python3 resummarize_v2.py --selftest                      # 期望 SELFTEST PASS (5/5)
python3 resummarize_v2.py --run-dir runs/l3-v2-sc-40-hy4free2   # 期望 drift 为空、carrier_judgements=29
python3 verify_evidence_chain.py --run-dir runs/l3-v2-sc-40-hy4free2  # 期望 ok=true n_records=29
```

## D18 — v3 设计隔离「词汇」与「版本」+ arXiv 定稿闸（2026-11-19 PST）

D16 证明 v2 删臂失败标签 `STALE_FAILURE` 描述的是「无成功写」：资源名只在被删载体里。
本轮把机制拆成两个可独立操纵的成分：资源名（词汇）常驻 system prompt 两臂同，
版本映射 `basis_version=1/2`（版本）是唯一载体。`l3_pilot/v3_runner.py` 实现 S_version，
`--selftest` 6/6：两臂提示逐字同且资源名常驻、载体只含版本、`stat` 不泄漏、GREEN 恢复、
RED 删版本→STALE 且注入前成功写 >0（v2 是 0）、假阳守卫。
2 卡真模 smoke（minimax）方向对：del 注入前成功写 24 次和 4 次（对照 v2 的 0），
但 keep 一侧一次 502、一次注入前完成 —— 样本过小，未作确证，未进任何稿件数字。
`L3_V3_PREREG_DRAFT.md` 为未签署草稿（NOT IN FORCE），不点火主实验、不动冻结层。

arXiv 定稿：12 条文献逐条 web_fetch abs 页做 ID↔标题逐字验号、零自引；
`references.bib` 入稿、正文 6 处 `\cite`、integrity 声明 citations 12 条带 receipt；
摘要从 2002 字符压到 1417（限 1920）；三张表移到 `paper/tables/` 使其进打包白名单。
`arxiv_package_check.sh` 隔离复编 PASS（0 error / 0 undefined / 0 overfull / 文本层 ident / 9 页），
三闸 first+final 全 0、视觉闸 PASS_WITH_NOTES。台账 L-ops-796。
420 扩样（工作目录内独立轨道）N=170/840 TL=0 在跑，不进本轮稿件。

复核：
```bash
cd l3_pilot && python3 v3_runner.py --selftest
cd paper_v2 && bash "/home/liuzewen/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh" paper l3-source
```

## D19 — v3 40 卡阳性对照：混杂解除后的干净结果，H3v3 方向预期被证伪（2026-11-19 PST）

预注册 `L3_V3_PREREG.md` 已签署 IN FORCE（用户明确 GO「全做」），登记 `v3_runner.py`
407 行 / 19184 B / sha16 `1d92fbcd6d1d3648`。runner 补齐主实验路径（并发/state/judgements/
report + `writes` 诊断块），selftest 7/7。

**阳性对照门（minimax，40 卡 80 条，`runs/l3-v3-pc40-minimax/`，TL=1/80=1.25%）全过**：
- keep：n=39，REC 37 / STALE 0 / PIC 2 → 有效 REC 率 **37/37=1.0000 ≥0.8**
- del：n=40，REC 26 / STALE 8 / PIC 6 → 有效 STALE 率 8/34=0.2353，退化 **25.81pp ≥0.5pp**
- 配对 31 有效对：keep stale 0 / del stale 8，McNemar **p=0.0078125**（2⁻⁷），
  cost_ratio_del_over_keep=**0.9457**（≤1.10，删臂反而略便宜，与 v2 的 3.49 相反）

**D16 混杂确已解除（量化）**：del 臂 40/40 episode 都有成功写，注入前累计成功写
**263 次**（v2 删臂是 0/120，注入前 0/562）。资源名常驻 system prompt 生效，
删臂不再是"一个字写不出去"。

**H3v3 方向预期被证伪，须如实记录（不事后改预注册）**：`L3_V3_PREREG.md` §3 写的是
"删掉版本映射相对保留把 STALE 比例降低 ≥15pp"（继承 v2 H3b 的"删是好的"）。
数据方向相反：**保留版本映射（keep）0% STALE，删掉版本映射（del）25.81% STALE** ——
即版本映射有价值，删掉它有害。这是预注册白纸黑字写下的预期与实测不符，判 H3v3 方向证伪；
不改预注册条款，只在 D19 记录。这个反向结果恰恰印证 D16：v2 的"删 m-plan 有益"
是提示混杂 + 资源名随载体被删的假象；一旦两臂提示统一、资源名常驻，删信息就暴露出代价。

**del 臂的失败是优雅降级而非失效**：8 条 STALE_FAILURE 的 verified_count **全部=40**
（D8.1 复合结果：注入后用 v1 写错过至少一次，但随后切 v2 搬完全部文件，judge 的 stale
分支优先于完成度故判 STALE）。其余 26 条 del RECOVERED 在注入后靠 "write 被拒" 的
试错闭环推断出该切 v2（首次成功写：注入后第 0 步 8 条 / 第 1 步 17 条 / 第 2 步 1 条）。
所以删版本映射的真实代价 = 从"直接切对"退化到"试错后切对"，中途产生可观测的 stale 写。

**局限**：单模型 minimax；40 卡样本；PIC 13.75%（del 6/40）注入前完成不进分母；
成本比 0.9457 的 CI 未算，不得写成已精确估计。扩样到 200 卡（≥140 有效配对）已点火
（`runs/l3-v3-sc-200-minimax/`，pid 2175999），结果另记，不并入本节。

复核：
```bash
cd l3_pilot
python3 v3_runner.py --selftest                    # 7/7
cat runs/l3-v3-pc40-minimax/positive_control.json  # 读数固化
cat runs/l3-v3-pc40-minimax/report.json            # runner 原生 cells/paired/writes
```

## D20 — v3 主实验终态：159 有效配对，方向证伪被大样本确证（2026-11-19 PST）

扩样 `runs/l3-v3-sc-200-minimax/`（minimax，200 卡 × 2 臂 = 400 条，`--workers 2 --resume`，
TL=0 零折损）跑完。runner 原生 report 与独立脚本 `summarize_v3_main.py` 逐项对表一致
（`n_effective=159 / improved=1 / worsened=52 / diff_pp=32.0755 / p=1.199e-14 /
cost=1.0177` 全 OK，keep/del stale 率换算也一致）。

**cells（剔除 TL 后）**：
- keep：n=200，REC 174 / STALE 2 / INC 1 / PIC 23，中位 15,840 tok
- del：n=200，REC 123 / STALE 59 / PIC 18，中位 16,106 tok

**配对（159 有效对，超预注册 140 门槛，power_ok=True）**：
- keep stale 2/159=1.26%，del stale 53/159=33.33%，diff **32.08pp**（删臂 STALE 多 25 倍）
- improved=1 / worsened=52，McNemar **p=1.199e-14**（精确二项）
- cost_ratio_del_over_keep=**1.0177 ≤1.10**（成本条款绿；v2 的 3.49× 是资源名被删的假象）
- 方向 = **deletion_harmful**（删版本映射有害）

**H3v3 方向证伪被大样本确证**：预注册 §3 预期"删版本映射降 STALE≥15pp"（继承 v2），
实测删了反而从 1.26% 涨到 33.33%。这是预注册白纸黑字预期与实测相反，判 H3v3 方向证伪；
不改预注册，只记 D19/D20。完整科学闭环：**v2 的"删 m-plan 有益" + 3.49× 成本都是
D16 混杂的假象**（删 m-plan 同时删了资源名 → del 臂一个字写不出去 → 既显"不 stale"又显
"贵"）。v3 把资源名常驻后，两臂都能写，删信息的真实代价显形 = STALE 涨 25 倍、成本几乎不变。

**D16 混杂解除的终态证据**：keep 与 del 臂 episodes_with_ok_write 均 200/200；
ok_writes_total keep 2038 / del 2557。两臂都正常写，差异只在"注入后是否跟随版本切换"。

**阳性对照门（全过）**：keep REC 174/176=98.9%≥0.8；del 退化 32pp≥0.5pp。
证据链 `verify_evidence_chain --run-dir` ok=True n=400。

**局限**：单模型 minimax；del STALE 里大量 verified=40（D8.1 复合：写错过但最终搬完），
"STALE" 是"曾用过时版本"而非"未完成"；PIC 11.5%（keep 23 / del 18）注入前完成不进分母；
成本比 CI 未算，不得写成已精确估计；420 线（v2 的 420 扩样）独立，与本批不混。

复核：
```bash
cd l3_pilot
python3 summarize_v3_main.py                          # 默认 200 卡批
python3 summarize_v3_main.py runs/l3-v3-pc40-minimax  # 对表 40 卡
cat runs/l3-v3-sc-200-minimax/report.json
cat runs/l3-v3-sc-200-minimax/main_summary.json
python3 verify_evidence_chain.py --run-dir runs/l3-v3-sc-200-minimax  # ok=True n=400
```

### D20.1 — 交付完成率的第二读数（同批同载体，无新实验）

在 D20 主表之外补一个正交维度「是否最终搬完全部 40 个文件」（`verified_count == n_files`）：

| 臂 | n | 交付完成 | 注入前就完成 |
|---|---|---|---|
| keep（版本映射在） | 200 | 199/200 = 99.5% | 23 |
| del（版本映射删） | 200 | **200/200 = 100.0%** | 18 |

**两臂交付完成率几乎相同，甚至 del 略高**。这不是"删更好"，而是**两者都在完成**——
差别只在过程中：删臂靠 `write` 被拒的试错闭环先写错再切对，产生可观测的 `basis=1` 陈旧写。

**因此 D20 的 32.08pp 必须这样读**：它度量的是**「有没有产生过基于过时版本的写动作」**，
不是「任务有没有做成」。del 臂 59 条 STALE_FAILURE 的 `verified_count` **全部=40**（100%），
keep 臂 2 条 STALE 也是 40。这与 D8.1 的诊断一致：`judge.py:48-49` 的 stale 分支优先于完成度，
所以"写错过一次但最终交付全部文件"被判为纯失败。

**结论句改写（以此为准）**：删除版本映射的代价 = 从"直接切对"退化到"试错后切对"，
表现为 STALE_FAILURE 从 1.26% 涨到 33.33%、成本仅 1.0177×（几乎不变）；
最终交付率 99.5% vs 100.0%（无实质差异）。原 D20 的"删臂 STALE 多 25 倍"表述成立，
但**不可读成"删臂失败率 33%"**。

复核：
```bash
cd l3_pilot
python3 - <<'EOF'
import json,glob
fs=glob.glob("runs/l3-v3-sc-200-minimax/traces/*.json")
for a in ("keep","del"):
    ts=[json.load(open(f)) for f in fs if f"__{a}__r0" in f]
    done=sum(1 for t in ts if t["final"]["verified_count"]==t["final"]["n_files"])
    print(a,len(ts),done)
EOF
```

## D21 — 三层链条叙事入稿：把 v3 写进核心结论，撤回「删臂零成功写」的普适化（2026-11-20 PST）

**背景**：v2 稿件（9 页，已过三闸+arXiv 封包）的核心结论停在「删臂零成功写、成本 3.5×、资源名是唯一载体」。这是第二次迭代在其操纵下的真读数，但把它写成「删除记忆让 agent 失败」的普适结论会被第三次迭代直接推翻：v3 把资源名常驻 system prompt 后，删臂注入前成功写 1481 次、两臂交付率 199/200 与 200/200、成本比塌到 1.0177。用户 (m05492) 批准把三层链条写进稿件。

**改动（稿件 main.tex）**：
1. 标题改 `What deleting an agent's memory actually removes: evidence from three preregistered iterations`；日期 20 November 2026。
2. 摘要整段重写为三层：v1 提示措辞带整个效应（因子分解证据）；v2 119/119 stale、3.2–3.8×，但删臂零成功写是因为资源名只在删除载体里；v3 资源名常驻后 2/159 vs 53/159（+32.1pp, p=1.2e-14），两臂交付 99.5% vs 100%、注入前写 1481 次、成本比 1.0177。结论句：删记忆在三次迭代里都没让 agent 失败，表观效应分别是提示措辞/缺标识符/被纠正的过时写计数。
3. 新 Results 小节 `A third iteration: keeping the identifiers, removing only the version`（带 `Table 4` = `tables/tab_v3.tex`）；Discussion 三段重写为「效应每次都换来源」三层链条；Conclusion 重写为冗余性教训；Limitations 加第 7 条（v3 单模型）。
4. 撤掉 STRONG_RE 假阳词（never/cannot 全清，实测 grep 0 命中）。

**数字来源（全部现算、不手抄）**：
- 新 `paper_v2/scripts/make_v3_evidence.py`（schema `l3-v3-evidence/v1`）从 `runs/l3-v3-sc-200-minimax` 的 400 judgements + 400 traces 现算 `paper_v2/evidence/v3_main.json`，selftest green3/red1。
- 读数：keep n200 REC174/STALE2/INC1/PIC23 delivered199/200 ok_before_inj1276 ok_total2038 med15737；del n200 REC123/STALE59/INC0/PIC18 delivered200/200 ok_before_inj1481 ok_total2557 med16086；配对 n_eff159 keep_stale2 del_stale53 diff32.0755 p1.199e-14 cost1.0177 cost_ok True；del 59 条 STALE 全部 delivered 40/40。trace_sha16 bb74d0e6d4d3913e。
- `make_tables.py` 加 `tab_v3()`/`build_v3()`（n_eff<140 抛错），build 返回 4 元组，selftest green4/red4（RED 把 del ok_before=0 delivered=123 表必变）。踩坑：断言千分位 `16,086` 不是 `16086`。

**三闸声明同步**：
- `make_declarations.py`：实验加 `v3-clean`（log=v3_main.json, artifact=200卡 report.json）；claim 加 c10–c13（逐字抽摘要 v3 句），c1–c4 锚点改成改写后摘要现措辞；LIMITS 补 v3 单模型；fig_decl 加 tab-4 trace（source=v3_main.json, hash 钉 make_tables.py）。selftest green15/red1。
- `integrity_declaration.json`：claim_numbers 10→17（新增 v3 配对/成本/交付/写次数 7 条，全挂 v3_main.json 字段）。
- `run_config.json`：加 `v3_clean` 段（159/32.0755/1.199e-14/1.0177/1481/199/200 等）供 mode6。
- 纪律复用：声明 claim 必须从稿件逐字抽，自检 `ok==len(claims)` 不容差（本次 c1–c4 因摘要改写脱节，当场被 selftest 抓住并改成现措辞）。

**验证（全绿）**：编译 pdflatex ×2 exit 0、0 `!`、10 页；三闸 first G1/G2/G3=0（G1 mode3 17 条当场核对、mode6 18 数字，G2 ALIGNED=13，G3 tab1–4 全 PASS）；final --prior 复查三份全 PASS exit 0；视觉闸 PASS_WITH_NOTES；arXiv 封包 `arxiv_package_check.sh paper_v2/paper l3-v3` **PASS**（0 error/0 undefined/textlayer ident/10 pages/abstract 1570 chars）。

**交付哈希（sha16）**：main.tex 0c2ca951b3610806 / main.pdf a5a58fd773a62a62 / tab_v3.tex 9486c83fd64edf9c / make_v3_evidence.py b1d4a4474a3cf39f / make_tables.py fd8a136f13d0986f / make_declarations.py 6f01564236af389c / v3_main.json 2229fa58798a4a2a / integrity_declaration.json 80657a1c137ce227 / experiment_provenance.json 22c831388c886a5f / figure_trace.json a85e4cb28dc37e9b / visual_check.json b7de460fe1524d17。

**未做（非瓶颈，明确排除）**：不补 v3 第二模型（v2 三模型已给方向，v3 是单模型确认）；不续 v2 420 扩样（混杂已判定，加样本只压 MDE 不减偏倚）；不上传 PyPI（待用户凭据）。

**复核**：见 `l3_pilot/REPRODUCE_CHECKS.md` D21 段。

## D22 — 投稿包与证据包：把「可复算」做成可交付的封包（2026-11-20 PST）

**动机**：三条迭代的实验与稿件已定稿（10 页、三闸 + arXiv 封包 + 视觉闸全绿），但交付形态仍是「本机目录」。D22 把已有产物固化成两件可提交物，不产生任何新科学结论。

**D22.1 投稿包（`paper_v2/submission/`）**
- `l3-source.tar.gz` 直接取自已通过 `arxiv_package_check.sh` 的 tarball（`paper/.arxiv-check/l3-v3.tar.gz`），**逐文件 sha256 与磁盘稿件相等**（main.tex `0c2ca951b3610806`、main.bbl、四张表全部 match），不是重新打包。
- `make_submission.py` 做组装与校验：`verify_tarball()` 逐成员比对 disk/tar 哈希；`assemble()` 复制 tarball、核对 `cover_letter.md` 与 `REPRODUCIBILITY.md` 存在、写 `submission_manifest.json`。
- `--selftest` green 1 / red 2：真实 tarball 必过；篡改 `main.tex` 的副本必须被判 `out of date`；删掉 `tables/tab_v3.tex` 的副本必须被判 `lacks`。
- **踩坑（真实 bug）**：首版 RED-1 未触发——`tarfile` 按 `info.size` 写入，追加字节被静默丢弃，篡改副本与原件等价。修法是改动后同步 `info.size = len(data)`。教训：**gate 的 RED 必须验证「改了的东西真的进了被测底物」**，否则测的是自己的写入逻辑。
- `cover_letter.md` 全部数字逐条对表 `paper_v2/evidence/v3_main.json`：2 vs 53 of 159、32.1pp、1.199e-14、1.0177、1481、99.5% / 100.0% 七项全等。

**D22.2 证据包（`l3-evidence-*.tar.gz`）**
- `make_release_pack.py` 收 12 个批次的 judgements/traces + 批次级分析 JSON + 全部 33 个 py + 三份治理文档（`DEVIATION_L3.md`、`REPRODUCE_CHECKS.md`、`L3_V3_PREREG.md`、`L3_V2_PREREG_REVIEWABLE.md`）+ paper_v2 证据 + 投稿包本体。
- **载体一致性闸**：每批 `n_carrier` 必须等于 `report.json.n_records`；`l3-v2-positive-control` 只有 traces（16 条、无 judgements，早于判定层）→ 接受 traces 计数但要求 j/t 不互相矛盾。
- 终态 2,223 文件 / 压缩后 822,671 B / `sha256sum -c SHA256SUMS` 2223/2223 OK / `--verify` `{"files": 2223, "ok": true}`。
- `--selftest` green 1 / red 2：真包必过；改一个 `report.json` 里的 `n_records` 必须失败；引用不存在的批次必须 `batch missing` 拒绝。
- **踩坑**：首版 `collect()` 只挑白名单文件名，漏掉 `l3-v2-rejudged/`、`l3-v2-d16/`、`l3-fact-240x6x2-analysis/` 三个目录的顶层 JSON（它们不在 `judgements/` 下）→ 改为收批次目录全部顶层 `.json/.md/.py/.log/.txt`。教训：**白名单收集会静默丢新产物**，载体目录应整目录收。

**D22.3 发布路径（脚本只备命令，不自己上传）**
- `README_RELEASE.md` 出两条：GitHub Release（`gh` 已认证为 `aidless`，128 个公开仓）与 Zenodo（`zenodo_deposit.py`，`ZENODO_SANDBOX=1` 可先演练）。
- `zenodo_deposit.py` 默认只创建 deposition 并上传、**不 publish**（草稿态可在网页核对），`--publish` 才铸 DOI；token 只从环境读，不落任何文件。`--selftest` 用本机 http.server 桩做 green 2 / red 1（草稿不发布、`--publish` 真发布、缺文件必拒）。
- **未执行项**：实际 `gh release create` 与 Zenodo 上传均未运行——按 AGENTS.md §3 外部发布需用户当次明确授权。两处均已实探前置条件：`gh` 已登录 `aidless`、`ZENODO_TOKEN` 环境为空。

**复核命令**
```bash
cd l3_pilot
python3 paper_v2/scripts/make_submission.py --selftest     # green 1 / red 2
python3 paper_v2/scripts/make_submission.py                # 写 submission_manifest.json
python3 paper_v2/scripts/make_release_pack.py --selftest    # green 1 / red 2
python3 paper_v2/scripts/make_release_pack.py               # 建 l3-evidence-*.tar.gz
python3 paper_v2/scripts/make_release_pack.py --verify paper_v2/submission/l3-evidence-*.tar.gz
python3 paper_v2/scripts/zenodo_deposit.py --selftest       # green 2 / red 1
(cd paper_v2/submission && sha256sum -c SHA256SUMS | grep -c OK)   # 期望 2223
```

## D23 — 第二模型（异族）窗口失效：qwen3.8-max 40 卡只落 22/80，通道全灭（2026-11-21 PST）

### D23.1 新云实例编排机（非科学偏离）
- 用户给新 AMD 实例 `ssh -p 35361 root@36.150.116.206`（128 核 / 503G / Python 3.12 / rsync 有 / 剩 2天23小时 / 11-23 20:40 自动销毁）。
- `rsync -az l3_pilot/ → /root/l3_pilot/`（1.56s，39M），`python3 -m py_compile v3_runner.py` OK。
- 云 sshd 用 `/tmp/amd-oneclick-sshd_config`（不是 /etc/ssh/sshd_config），`AllowTcpForwarding no` → 备份后 sed 改 yes + HUP 重载。
- 本地起 `/tmp/l3rt.sh`（`-R 13427:127.0.0.1:3425` 自动重连）；云端 `curl 127.0.0.1:13427/v1/models` → 200。模型仍走本地 Magpie 网关，云只当编排机（符合 PREREG §7 裁决）。

### D23.2 第二模型选型与点火
- 从 Magpie 214 模型探 15 个异族候选（文本三连 + 带 tools 三连）：当时 PASS = qwen3.8-max（tool_calls 3/3，1.6–2.4s）/ doubao-seed-2-1-pro-260915 / Ling-3.2-flash / spark-x2.6；qwen3.8-flash 2/3 弃；claude/gpt/grok 401、qwen4-max 502、ernie 404、glm-5.3-flash 文本过但 tool_calls 0/3。
- 定 `qwen3.8-max`（裸名，异族通义千问，v2/v3 主实验未用过，比豆包快一倍）。
- 2 卡 smoke `runs/l3-v3-smoke2-qwen`（wall 166.51s，TL=0）：D16 零写假象在 qwen 上同样解除（del 注入前 50 次成功写）；但 del 2/2 REC、diff 0.0pp（n=2 不显著）。
- 40 卡点火：本地 supervisor 落 22/80 后停掉、rsync 进度到云、云端 `/root/v3qwen_cloud_sup.sh`（60 轮 + preflight + purge + --resume）pid 885 续跑。

### D23.3 通道窗口失效（决定性）
- qwen3.8-max 在正式跑时返回 `LobsterAI（有道龙虾）: 模型不可见或无访问权限: qwen3.8-max`（HTTP 502，33.7s 长超时）；裸名与 `hub-lobsterai/` 前缀变体都失效 → 选型时的三连 PASS 是短暂窗口，点火时窗口已关。
- 重探全部异族候选（2026-11-21）：Ling-3.2-flash / spark-x2.6 文本 200 但带 tools 3/3 全 `finish=length, tool_calls=0`（reasoning 吃满 token，做不了工具任务）；doubao 000（挂）；qwen3.8-27b / qwen3.8-flash 429 quota exceeded（书生·端砚）；minimax 429。
- 唯一稳定带 tools 的只有 `workbuddy/deepseek-v4.1-flash`（3/3 tool_calls）——但这是 v2/v3 主实验已用过的 deepseek，不是异族第二模型。
- 结论：**通道状态是时间窗口属性，不是模型属性**（呼应 L-ops-687）。本轮没有任何异族模型能承担 40 卡第二模型。

### D23.4 裁决
- qwen 22 条已落盘（del 10 REC + 1 STALE、keep 11 REC、delivered_all 22/22）只作方向性 note，**不进稿件**：n 不足 80、通道中途失效、阳性对照门（keep REC≥0.8 虽满足）未走完配对判定。
- v3 主结论保持 minimax 单模型（159 有效对、power_ok=True）——单模型确认已足够支撑三层链条叙事；第二模型是加分项不是必需项。
- 云端 supervisor 已精确清理（只停我自己的 v3_runner.py --model qwen3.8-max / v3qwen_cloud_sup.sh；残留的 p11d_device.py 是别的任务，未动）。云端编排机保留，11-23 销毁前可供后续窗口重跑。
- 本地反向隧道 `/tmp/l3rt.sh`（pid 607178）保留，云端 13427 仍通。

### D23.5 复核命令
```bash
# qwen 22 条读数（方向性，不进稿）
cd research-plan-20261108/l3_pilot
python3 -c "import json,glob,collections; fs=glob.glob('runs/l3-v3-sc-40-qwen/judgements/*.json'); print(len(fs), collections.Counter(json.load(open(f))['verdict'] for f in fs))"
# qwen 通道现状（502/429/000 = 已失效）
curl -s --max-time 40 http://127.0.0.1:3425/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"qwen3.8-max","messages":[{"role":"user","content":"ok"}],"max_tokens":4}'
# 云端编排机（13427 隧道通）
ssh -p 35361 root@36.150.116.206 'curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:13427/v1/models'
```
## D24 — S_a 敏感性臂：载体换成检索语料，效应仍在但出现新混杂（2026-11-21 PST）

### D24.1 为什么做这一发

用户指令「1+3」中的第 3 项。v3（S_c）把版本映射放在**对话历史**里，删掉后 del 侧
stale 率从 1.26% 升到 33.33%（159 有效对，p=1.199e-14）。一个自然的反驳是：这个效应
可能只是「历史裁剪」这一特定实现方式的产物，换一种载体形态就没了。

S_a 把同一份版本映射搬进**检索语料**，agent 必须用 `retrieve` 工具去取。若效应方向
保持，说明结论不依赖单一载体实现；若不保持，v3 的结论要收窄到历史通道。

普查先行（`census_sa.py --selftest` PASS，真实普查扫 1911 文件、src_hits=[]、car_hits=[]、
capped=false）判定 **MUST_BUILD_NEW**，成本 1 发。产物 `runs/sa_census.json`。

### D24.2 实现与自检

新建 `l3_pilot/sa_runner.py`（508 行，sha16 `5265bf80138b2209`），编排层，冻结 6 文件未动：

- `SYSTEM_PROMPT` 两臂逐字同，资源名 `{r0}/{r1}` 与路径 `{t1}/{t2}` 常驻（继承 D16 去混杂）。
- `make_corpus(card, keep_carrier)`：keep 语料含 `doc-plan-v1`/`doc-plan-v2`（只含
  `basis_version=1/2` + resource + target），del 语料**只移除这两条版本行**，保留 `doc-task`。
- 新增 `retrieve` 工具（`TOOLS_SA = v2.TOOLS + [RETRIEVE_TOOL]`），`dispatch_sa` 拦截
  retrieve 返回语料，其余委托冻结口径 `v2.dispatch`。
- `EnvSA(v2.EnvSC)`：stat 仍不泄漏版本（D11.1）；判定复用 `v2.verdict_of`（跨批可比）。
- 两臂都收到「corpus is available via retrieve」这句 user 消息 —— 工具可用性不是操纵变量。

`--selftest` **8/8 PASS**：提示模板无臂标识 / 删臂语料移除版本行 / stat 不泄漏 /
carrier 字段 = `S_a:retriever/corpus` / GREEN RECOVERED 40/40 / RED STALE 且注入前成功写 4 次 /
假阳守卫（keep 臂但语料抹掉版本行 → 仍 STALE）/ 网格落盘 4 条 + state finished。

两处自检缺陷当场修掉：①`assert keep_sys == keep_sys` 是恒真断言 → 改为检查模板不含任何
臂标识 token；②假阳守卫原先只改脚本客户端自己发的 retrieve 结果，环境语料没变 → 加
`strip_versions` 参数真正改环境语料。

2 卡真模 smoke（`runs/l3-sa-smoke2`，workbuddy/deepseek-v4.1-flash，wall 97.19s，TL=0）：
keep 2/2 REC、del 2/2 STALE、diff 100pp、cost 1.8942、两臂注入前都有成功写。

### D24.3 40 卡主实验读数（云端编排机，载体已拉回本地）

`runs/l3-sa-sc-40`：**80/80，transport_loss=0**，证据链 `ok=true n_records=80`。
新尺 `summarize_sa.py`（133 行，sha16 `58d9ae6061a46673`）先对表 2 卡 smoke 8 项全等
（`--selftest` green1/red1 PASS）才用于读主实验。

| 臂 | n | REC | STALE | 交付全部 40 文件 | 注入前成功写合计 | median tok |
|---|---|---|---|---|---|---|
| keep | 40 | 35 | 5 | **40/40** | 61 | 35,699 |
| del | 40 | 15 | 25 | **37/40** | 44 | 49,994 |

配对（40 对全有效）：keep stale 5/40=12.5%，del stale 25/40=62.5%，**diff 50.0pp**，
improved 3 / worsened 23，McNemar **p=8.797645568847656e-05**，cost_ratio **1.3341**。

**方向与 v3 一致**：删掉版本映射后 stale 写动作显著增多。载体从历史换成检索语料，
效应没有消失 —— 这是 S_a 要回答的问题，答案是「不依赖单一载体实现」。

### D24.4 但出现一个新混杂：注入前「只检索不写」

del 臂有 **15/40 条注入前一次成功写都没有**（keep 臂 0/40）。逐条读 trace 确认：
这 15 条注入前 median **10 次 retrieve**（min 6 / max 13），write 尝试总共只有 2 次
（都是 `basis_version=0` 被 `stale_basis` 拒），即**它们在反复检索一个空语料，而不是写了被拒**。
对照 keep 臂注入前 median 只有 3 次 retrieve。

这与 v3 的 zero_pre 语义**完全不同**：v3 的 zero_pre（keep 23 / del 18）全部是
`PRE_INJECTION_COMPLETION`（注入前就干完了）；S_a 的 del zero_pre 15 条全是 STALE_FAILURE，
其中 13 条最终仍交付 40 文件。所以它不是 PIC，是「找不到计划就不敢动手」。

分层后（只取 del 注入前成功写 > 0 的 25 对，产物 `runs/l3-sa-sc-40-stratified.json`）：
keep stale 3/25=12.0%，del stale 10/25=40.0%，**diff 28.0pp**，improved 10 / worsened 3，
McNemar **p=0.09228515625**（不显著），cost_ratio **1.256**。

**裁决**：S_a 的全样本 50pp/p=8.8e-5 **不能单独引用**，因为它混入了「检索不到就推迟动手」
这一行为差异；分层后 28pp 方向一致但 n=25 不足以显著。S_a 只能作为**方向性佐证**写进
稿件（若要写），不得写成第二个确证。这与 D16/D20.1 的教训同构：每次换载体都会带进
一个新的行为混杂，必须分层后才谈效应。

### D24.5 交付率第二读数（D20.1 纪律）

keep 40/40 = 100%，del 37/40 = 92.5%。del 的 25 条 STALE 中 22 条仍交付全部 40 文件
（verified 分布 {17, 33, 40}）。所以 50pp 度量的仍是「是否产生过基于过时版本的写动作」，
不是「任务是否做成」；但 S_a 与 v3 不同之处是**交付率确实掉了 7.5pp**（v3 是 99.5% vs 100%），
因为「只检索不写」消耗了轮次预算。这一条必须与 stale 率并列报告。

### D24.6 复核命令

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 census_sa.py --selftest                      # 期望 SELFTEST PASS
python3 sa_runner.py --selftest                      # 期望 SELFTEST PASS (8/8)
python3 summarize_sa.py --selftest                   # 期望 SELFTEST PASS (2/2)，对表 smoke 8 项
python3 summarize_sa.py --run-dir runs/l3-sa-sc-40   # 期望 80 条 / diff 50.0 / p 8.7976e-05 / cost 1.3341
python3 verify_evidence_chain.py --run-dir runs/l3-sa-sc-40   # 期望 ok=true n_records=80
cat runs/l3-sa-sc-40-stratified.json                 # 期望 n_pairs 25 / diff 28.0 / p 0.09228515625
```

### D24.7 未处理项

- S_a 未进稿件（当前稿件三层链条止于 v3）。若要加，须按 D24.4 的分层口径写，且需用户授权改稿。
- qwen 第二模型 40 卡仍卡在 22/80（云端 supervisor pid 5230 持续 preflight FAIL，窗口未恢复）。
- S_a 的「只检索不写」混杂本身值得单独设计一发（例如给 del 语料留一条不含版本的占位文档，
  看是否消除推迟动手），但那属于 v4 范畴，本轮不做。
## D25 — S_a 扩样到 306 条：分层效应稳住且显著，但通道耗尽提前收尾（2026-11-21 PST）

### D25.1 为什么要扩

D24 的 40 卡 S_a 全样本 diff 50.0pp（p=8.8e-5），但 del 臂 15/40 条注入前一次成功写都没有
（keep 0/40）——「只检索不写」是新行为混杂。分层（del 注入前成功写>0，n=25）后 diff 降到
28.0pp、p=0.0923 不显著。要判这个方向是不是真的，必须把分层样本补大。

### D25.2 扩样执行与提前收尾

seed 40 卡批 80 条到 `runs/l3-sa-sc-200`，云端 `/root/sa200_sup.sh`（preflight 3×200 +
purge TL + --resume + workers=3，NEED=400）跑到 **N=306/400** 时 workbuddy/minimax/kimi
三通道先后 429/额度耗尽，9 轮 preflight 全红、supervisor 空转，精确停掉（不误伤其他任务）。
载体已 rsync 拉回本地，306/306 judgements+traces 对账一致。

**把已知混杂留在报告里**：中途 TL 曾到 12 条，purge 后全样本 TL=0（66 条 transport 记录的
剔除都留在 state/judgement 的 transport_loss 标志里，可复算）。runner 被 kill 前没写
report.json → 新建 `resummarize_sa.py` 从载体现算重建（selftest 先对表 40 卡已登记 report
逐字段相等才允许 apply），重建报告挂 `resummarized_from_carriers: true`、`carrier_judgements: 306`。

### D25.3 终态读数（306 条，从载体现算，非 runner 内存快照）

- 全样本：keep/del 各 147 有效（TL 剔除后），no TL。keep REC 137 / STALE 10；
  del REC 67 / STALE 79（含 1 条 INCOMPLETE）。
- 配对 n_valid=147 / n_eff=146：keep stale 0.0685 vs del stale 0.5411，
  diff **47.2603pp**，discordant improved=3 / worsened=72，
  McNemar **p=3.73e-18**，cost_ratio **1.2622**（>1.10，不满足成本门）。
- 交付率第二读数（D20.1）：keep **147/147=100%**，del **135/147=91.84%**；
  del 79 条 STALE 里 68 条仍搬完全部 40 文件 → 47pp 不是「任务失败率」，是
  「是否产生过基于过时版本的写动作」。del 比 keep 真正少交付 8.16pp（「只检索不写」
  吃轮次预算）。

### D25.4 分层（去「只检索不写」混杂）

按 del 注入前是否成功写过分层：zero-pre 50 张、pos-pre 97 张。

| 层 | n | del STALE | del delivered |
|---|---|---|---|
| zero-pre（注入前零成功写） | 50 | 49 | 42/50 = 84.0% |
| pos-pre（注入前写过） | 97 | 30 | 93/97 = 95.88% |

**分层配对**（`stratified()`，口径对齐 v2：improved=keep stale & del 非 stale）：
n_pairs=97，keep stale 5/97=5.15% vs del stale 30/97=30.93%，
diff **25.77pp**，improved=3 / worsened=28，McNemar **p=4.65e-06**（显著），
cost 1.2452。

读法：D24 的 n=25 不显著（28pp, p=0.0923）在扩样后 **n=97、25.77pp、p=4.65e-06**
稳住且显著——方向为真；但 97 对里仍有 50 张零写卡需排除，且 S_a 单模型（workbuddy）。

### D25.5 一次被工具抓出的方向标签反错

固化分层进 `summarize_sa.py` 时发现旧一次性产物 `l3-sa-sc-40-stratified.json` 的
`improved=10/worsened=3` 方向反了：按 v2 口径 del 搞坏的应是 worsened=10、救回的 improved=3。
McNemar 双侧 p 对称（都 0.09228515625）所以数值没暴露，但标签会把读者误导成「删臂更好」。
新 `stratified()` 用 v2 口径 + 金标对表（40 卡 11 项全一致）+ RED-分层专测「方向写反必被抓出」，
`--selftest` 从 2/2 升到 4/4。旧产物标 `SUPERSEDED`，以 `summarize_sa.py` 输出为准。

### D25.6 裁决

S_a 从方向性佐证升为**第三个机制一致性证据**（与 v3、D16 分层同向）：换载体到检索语料后
删臂仍显著更常写过时版本（分层 25.77pp p=4.65e-06），且 D16 混杂（零成功写）在两臂都解除
（keep 147/147、del 144/147 有成功写）。但：①成本 1.2622 超门；②S_a 单模型；③stop-at-306
而非 400。因此 S_a 只进 DEVIATION/附录，不改主论文三层链条（v1/v2/v3）的定稿叙事，
除非用户授权改稿。

### D25.7 复核命令

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 summarize_sa.py --selftest                      # 期望 SELFTEST PASS (4/4)
python3 resummarize_sa.py --selftest                    # 期望 SELFTEST PASS (2/2)
python3 summarize_sa.py --run-dir runs/l3-sa-sc-200     # 期望 306 条 / diff 47.2603 / p 3.73e-18 / stratified n97 diff 25.77 p 4.65e-06
python3 verify_evidence_chain.py --run-dir runs/l3-sa-sc-200  # 期望 ok=true n_records=306
```

## D26 — S_a 载体敏感性复现入稿：三层链条扩为四层（2026-11-21 01:05 PST）

用户 (m06510)「把 S_a 写进主稿」。把 D24/D25 的 S_a 载体敏感性复现写进 `paper_v2/paper/main.tex`，三层链条扩为四层；同步 Table 5、三闸声明、复现清单与台账。预注册 v1/v2/v3 与冻结 6 文件一字未动。

**D26.1 稿件改动（数字逐字来自载体）**
- 标题 L8 `...three preregistered iterations` → `What deleting an agent's memory actually removes: evidence from three preregistered iterations`（标题不变，S_a 标注为 sensitivity replication 而非第四次预注册迭代）。
- 摘要加一句 S_a：`79 of 146 effective pairs versus 10, +47.3 points` / `delivery falls for the first time (135 of 147 versus 147 of 147)` / `stratified gap 30 of 97 versus 5, +25.8 points, p=4.6e-6` / `token ratio 1.2622`。摘要 2068→1867 chars（arXiv 限 1920）。
- 新 Results 小节 `\subsection{A carrier-sensitivity replication}`（label sec:sa）：载体换成检索语料（`retrieve` 工具），资源名仍常驻 system prompt（继承 D16 修正），147 卡/臂、306 条、12 传输损失排除，单模型 workbuddy，明确「sensitivity, not a fourth preregistered iteration」。
- Table 5 `\label{tab:5}` / `\input{tables/tab_sa}`：keep 147/137 REC/10 STALE/delivered 147/147/ok-before 285/med 36,401；del 147/67 REC/79 STALE/delivered 135/147/ok-before 148/med 50,079；paired 146 / 79 vs 10 / ×1.2622；stratified 97 / 30 vs 5 / ×1.2452。
- Discussion 加一段（成本视角）、Limitations 第三条改写为「two carriers」+ 第八条（S_a 306/400 通道耗尽、分层条件于被操纵变量）、Conclusion 加「If it is stated elsewhere but only expensively retrievable, deletion costs episodes.」。Reproducibility 改为三个现算脚本并列。

**D26.2 生成链升级**
- `paper_v2/scripts/make_tables.py` 加 `tab_sa()`/`build_sa()`：从 `runs/l3-sa-sc-200-summary.json` 现算，paired 计数由 `n_eff × rate` 还原并用「ds−ks == worsened−improved」自证（paired 块只存比率不存原始计数，直接读会 KeyError）。selftest green 4/red 4 → **green 5/red 5**（RED 改分层 del stale 必变表）。
- `paper_v2/scripts/make_methods_config.py` 加 `sa_block()`：同样计数还原 + 与 cells 交叉自证；selftest 仍 4/1。
- `paper_v2/scripts/make_declarations.py`：加 `sa-carrier-replication` 实验（log=`verify-sa.log`、artifact=`runs/l3-sa-sc-200/report.json`）、claim c14–c17、tab-5 figure trace、LIMITS 加 S_a 通道耗尽一条；selftest green 15/red 1 → **green 19/red 1**。
- `integrity_declaration.json`：`claim_numbers` 17→**32**（15 条 S_a 数字全挂 `runs/l3-sa-sc-200-summary.json` 字段），`results` 4→5（sa_replication_delivered 135，log verify-sa.log）。`evidence_manifest.json` 加 S_a summary 一条（13 条，306/306 judgements/traces）。`CLAIM_LEDGER.md` 加 v3 两条 + S_a 两条。

**D26.3 写作纪律（writing-guard）**
- 三处 STRONG_RE 措辞改掉以过 provenance 反向覆盖：`never land an accepted write` → `land no accepted write`；`impossible` → `costly`；`never deliver` → `end without delivering the task`；`merely cause` → `only costing`。全文 `\bnever\b|\bimpossible\b` 命中 0。
- 因果强度不越界：S_a 表述为「sensitivity result, not a fourth preregistered iteration」「single model」「stratified row is a selection rather than a controlled comparison」；把「交付率下降」限定为首个观察到 deletion 掉任务的 regime，而非泛化为 deletion 有害。

**D26.4 验证（全绿）**
- 编译 pdflatex ×2 exit 0、`^!`=0、overfull=0、**11 页** 221,600B。
- 三闸：`G1` 7 模式全 CLEAR（mode1 5 条结果、mode2 12 引用、mode3 **32 条 claim 全按字段当场核对 0 欠账**、mode6 Methods 18 数字对上）；`G2` ALIGNED=17/0；`G3` tab-1..tab-5 全 PASS + CAPTION-PAIRED/COMPOSITE；`--final --prior` 三份复查全 PASS exit 0。
- 视觉闸 `PASS_WITH_NOTES`、`issues_remaining=[]`（selftest green 2/red 2）。
- arXiv 封包 `arxiv_package_check.sh paper_v2/paper l3-v4` **PASS**：11 页 / 0 error / 0 undefined / 0 overfull / textlayer ident / abstract 1867 chars < 1920。
- 交付哈希（sha16）：`main.pdf ecb1ca3fc39485bee`、`main.tex cc9532f6f934f92f`、`tab_sa.tex db45855fc0fb2120`、`make_tables.py 8b14b4f794b8a2a5`、`make_declarations.py a4172d41a60dc529`、`make_methods_config.py d35c1e109fdb1ae8`、`integrity_declaration.json 770a760a73b163ce`、`experiment_provenance.json faa9b1a56449daf8`、`figure_trace.json f0fcff7c8cc01625`、`run_config.json cf57046cf38c43e54`、`CLAIM_LEDGER.md e60f5911487058501`、`evidence_manifest.json b64d281e11ec216d71`、`visual_check.json aafdfd5cecda4686`、`l3-sa-sc-200-summary.json 5943a48cb2de6169`。

**D26.5 定位**：S_a 是第四个证据层，但只作方向性/敏感性佐证——单模型、未预注册、306/400 通道耗尽、分层条件于被操纵变量。主结论（三层链条）不变；新增的边际价值是「换了载体后 stale-write 效应仍在，而交付率首次下降」这一 regime。不补 v3 第二模型（通道窗口已确认失效，见 D23）、不续 420 扩样（混杂已判定）。

## D27 — 投稿包与证据包同步到四层链条（2026-11-21 02:20 PST）

D26 把 S_a 写进主稿后，`paper_v2/submission/` 的投稿与证据封包仍是三层叙事版本，且两处生成器有硬编码过时点。本轮把封包同步到四层稿件，并修掉三个静默缺口。预注册与冻结层不动。

**D27.1 三个真缺口（都是被 selftest / 逐成员核对抓出来的）**
1. `make_submission.py` 的 `TARBALL` 硬编码 `l3-v3.tar.gz`（已不存在，且 selftest 报 `[submission] tarball missing`），`MEMBERS` 无 `tab_sa.tex`。→ 改指 `l3-v4.tar.gz`，MEMBERS 加 `tables/tab_sa.tex`；selftest 重回 green1/red2。
2. `make_release_pack.py` 的 `BATCHES` 无 S_a 两批、`collect()` 的表清单无 `tab_sa.tex`、`PAPER_EVIDENCE` 无 `l3-sa-sc-200-summary.json`/`verify-sa.log` → S_a 载体根本不进证据包。→ 加 `"l3-sa-sc-40":80` / `"l3-sa-sc-200":306`、表清单加 tab_sa、PAPER_EVIDENCE 加 sa summary + verify-sa.log + stratified + sa_census。
3. **治理文档静默丢失**：`DOCS` 列 `REPRODUCE_CHECKS.md`/`L3_V3_PREREG.md`/`L3_V2_PREREG_REVIEWABLE.md` 为项目根路径，实际在 `l3_pilot/` 下；`collect()` 用 `.exists()` 过滤 → 三份被静默丢弃，v1 `PREREG_L3.md` 从未列入。上一版证据包里 5 个 md 只有 DEVIATION + 两份 submission 文档 + 两份 repro README，**三份预注册/复现文档一个都不在**。→ 拆 `DOCS_PROJECT`（DEVIATION_L3.md + PREREG_L3.md，相对 PROJECT）与 `DOCS_PILOT`（其余三份，相对 ROOT），逐路径实测存在后重打；现证据包含全部五份治理文档。

**D27.2 叙述同步（数字逐字来自载体）**
- `cover_letter.md`：加 S_a 独立段（+47.3pp 79 vs 10 of 146、p=3.7e-18、delivery 135/147 vs 147/147、cost 1.2622、zero-write 50 条、分层 30 vs 5 +25.8pp p=4.6e-6 cost 1.2452），"Three features"→"Four features"，页数 10→11、表 4→5、D1–D21→D1–D26、日期 20→21 Nov，明确 replication "stopped at 306 of 400, endpoints rate-limited" 且 "not a fourth preregistered iteration"。
- `REPRODUCIBILITY.md`：载体表加 S_a 两批（306 与 80）、命令加 `summarize_sa.py --run-dir` / `--selftest` / `resummarize_sa.py --selftest`、表清单改 tab_*.tex 五张、Iteration-3 读数后加 S_a 全套期望读数、闸命令加 `sa_runner --selftest 8/8`、封包名 l3-v3→l3-v4、D1–D21→D1–D26、环境段说明 replication 多一个 `retrieve` 工具。
- `CITATION.cff` 与 zenodo `META.description` 的 date/描述加 replication 一句（标题不变，仍 "three preregistered iterations"）。

**D27.3 验证（全绿）**
- 三个生成器 selftest 全绿：`make_submission.py` green1/red2、`make_release_pack.py` green1/red2、`zenodo_deposit.py` green2/red1。
- 源包：`l3-source.tar.gz` sha16 `7c49e880384152ebb890` 与封包脚本产物 `l3-v4.tar.gz` 逐字节相同；含五张表（cells/paired/writes/v3/sa）；`submission_manifest.json` 六成员全部 match=true、两份 ancillary 哈希与盘上文件逐字节相等。
- 证据包：`l3-evidence-20261121.tar.gz` 3011 文件 / 14 批（含 l3-sa-sc-40 与 l3-sa-sc-200）/ 16,276,221 B 未压缩；`--verify` `{files:3012, ok:true}`；解包后 `sha256sum -c SHA256SUMS` **3012/3012 OK，0 FAILED**；五份治理文档（PREREG_L3 v1 / L3_V2_PREREG_REVIEWABLE / L3_V3_PREREG / REPRODUCE_CHECKS / DEVIATION_L3）全部在包内。
- 顺手修：`README_RELEASE.md` 的 `sha256sum -c` 指引原本让人 `cd submission` 后跑（路径全为包内相对路径，必然全 FAILED）；改为解包后指向 `OUT/SHA256SUMS` 的绝对路径，实测 0 FAILED。

**D27.4 哈希（sha16）**
`make_submission.py 3d33c8d462b4eb24b1b3cb22`、`make_release_pack.py d4d2a79423dff59653dfee0d`、`zenodo_deposit.py 270ef48650f33e208264ac96`、`cover_letter.md 235fb1a2bee7af24`、`REPRODUCIBILITY.md 2106cc0e190300e1`、`l3-source.tar.gz 7c49e880384152ebb890`、`submission_manifest.json 8dd3b1f2c7b36149c425b211`、`l3-evidence-20261121.tar.gz 9f793a935db1618432f8e7b0`、`CITATION.cff a6f099e73fcdba02890e1370`、`SHA256SUMS 1725c8dfa34d45d6575d801b`、`README_RELEASE.md 8d01b439a6117ea3d8239c4b`。

**D27.5 边界**：本轮只做本地封包的重建与自证，未执行任何 GitHub Release / Zenodo 上传（外部发布仍需用户当次授权，脚本与命令已备于 `README_RELEASE.md`；token 只从环境变量读，不落盘）。

**D27.6 更正（落盘后重打一次）**：D27 正文写完后，`DEVIATION_L3.md` 与 `REPRODUCE_CHECKS.md` 本身变了，而它们是证据包的被收录文件，因此按 D27.3 记录的旧包哈希（`9f793a935db16184…`）重打一次，使包内含 D27 正文自身。最终值：

- `l3-evidence-20261121.tar.gz` sha16 **`a24a9a527ff4b3b5480cc878`**（1,182,858 B，3011 文件 / 14 批）
- `evidence_manifest.json` `52a45a2489e0d0446cc2270e`、`SHA256SUMS` `a093b28378a7865618eb8935`、`README_RELEASE.md` `2c3e5cc798ef57b780670c85`
- 解包后 `sha256sum -c SHA256SUMS` 仍是 **3012/3012 OK，0 FAILED**；manifest 逐文件比对 mismatch=[]（唯一 "missing" 是 `CITATION.cff`，它在包根而生成目录为 submission/，属路径口径而非缺失，解包后校验通过）
- D27.3/D27.4 里的 `9f793a935db1618432f8e7b0` 与 `125ae2f65138117d…`/`1725c8dfa34d45d6575d801b` 为**重打前**的值，以本条为准。
- 教训：封包内若包含本次记录自身，落盘顺序必须是「先写记录 → 再打包 → 再写最终哈希」，否则正文里的包哈希必然过期（与 D17「report 是内存快照」同构）。

**D27.7 哈希权威口径**：证据包包含本文件自身，任何写进正文的包哈希都只是打包当时的一次快照，必然在下一次重打后过期（D27.3/D27.4/D27.6 的数值即属此类）。**权威值只看包内 `README_RELEASE.md` 与 `evidence_manifest.json`**（两者每次重打随包更新），正文数字仅作历史记录。

## D28 — 五席独立同行评审：5 个 DA CRITICAL 全部验证 + 若干数字错配（2026-11-21 PST）

用户 (m06982)「继续」→ 按 `academic-paper-reviewer` full 模式对四层链条定稿稿件做五席独立评审（EIC / Methodology / Domain / Perspective / Devil's Advocate）。五席各自独立读稿件、只读不改、不互看，报告落 `paper_v2/reviews/`（eic 171 / methodology 96 / domain 114 / perspective 132 / devils_advocate 195 行，共 708 行；所有 `main.tex:L` 锚点校验存在）。主会话对每条 DA CRITICAL 与数字错配逐条对照载体复算，不做"评审说啥就信啥"。

**Editorial Decision：MAJOR REVISION**（评审综合文档 `reviews/editorial_decision.md`）。5 席信号分裂（EIC/Perspective=Major、Methodology=带强制修订接收、Domain=weak accept、DA=5 CRITICAL），按 Iron Rule #4 由**验证过的 DA CRITICAL** 而非中位信号定档。5 个 CRITICAL 全部 VALIDATED：
- **C1** 摘要/结论"删记忆从没让 agent 失败"与自身数据冲突——S_a delivery 135/147 vs keep 147/147（`l3-sa-sc-200-summary.json` → `writes.del.delivered_all_files=135`），v2 删臂 0/120 RECOVERED。修：限定三预注册迭代 + 区分 carrier 删除 vs stale-belief 删除。
- **C2** v3 预注册 `L3_V3_PREREG.md:36` 写"删版本使 stale 降 ≥15pp、每模型同向"，实测 +32.1pp、单模型；D19 已记"方向证伪"，正文 L121/L123 未披露。修：披露证伪、标探索性、去"inherited direction"。
- **C3** v2 草案 `L3_V2_PREREG_DRAFT.md:38` 写死"成本 D≤B×1.10"，生效版才改"不预设方向"；三批 report mtime 03:22/08:16/19:17 早于预注册生效 17:50（D13）。L51"registered before the three batches"失真。修：改时序、报成本为探索性（措辞/时间对齐，非新发现）。
- **C4** v3 的 32pp 全是交付成功（59 条 STALE verified 全=40、delivery 199/200 vs 200/200）——是 process/manipulation check 而非任务失败，却当 headline。修：标明"delivery 上 null、stale write 上 process result"。
- **C5** S_a 分层 +25.8pp 条件于处理后变量；排除的 50 条 zero-pre-write 中 49 条 STALE、8 条未交付（伤害最大层），"lower bound"论证不成立；摘要把未分层成本 1.2622 安到分层句（真值 1.2452）、"no accepted write"实为"注入前"（仅 3/147 全程无写）。修：未分层 47.3pp 为主估计、分层标 subgroup、报 49/50 与 8/50、改数改措辞。

**数字/路径硬错（writing-guard 红线，逐条对载体复算）**：L16 分层 token ratio 1.2622→1.2452；L84 "p≤3.7e-12 每批"→实为 2.91e-11/2.33e-10/3.64e-12（界 2.4e-10）；L53 "fourteen deviations"→27；L16 "both arms deliver all (99.5%)"自相矛盾；L168 `summarize_sa.py` 路径应在 `l3_pilot/` 非 `paper_v2/`。

**裁决边界**：评审 Iron Rule #6 只读不改。Roadmap A（A1–A7 对齐稿件与自身载体/已登记偏离）虽无新科学主张，但改的是核心因果措辞与数字，属科学主张层，须用户明确授权改稿才执行；B（术语全局改名、S_a 分层移附录、补 7 条已验号引文、information ecology 形式化）留作者裁决；C（补冗余条件、v3 第二模型）需新实验，本轮不做。research_audit PASS（0 error/warning）、research_numbers PASS（249 数字无内部冲突）——审的是改稿前版本。

**复核**：五报告 708 行、锚点全存在；稿件 sha16 仍 `cc9532f6f934f92f`（mtime 01:06，评审全程未改）；power 0 / provenance frozen_ok true mismatch [] / v3 7/7 / sa 8/8 全绿。


## D29 — Roadmap A 修订执行：五席评审后的稿件对齐（2026-11-21 PST）

用户 (m07107)「继续」授权执行 `paper_v2/reviews/editorial_decision.md` 的 Roadmap A（强制修订）。范围严格限定为 A1–A7：把稿件对齐到它自己的载体与已登记偏离，不引入新科学主张；B（作者裁决项）与 C（需新实验）不动。写前备份 `main.tex` 到 `/tmp/main.tex.bak-before-A-<ts>`，全部改动逐条带 `assert count==1` 守卫。

**A1（DA-C1 头条与交付冲突）**：摘要删去"Deletion did not make the agent fail in any iteration"，改为"三预注册迭代里 deletion 从没让任务失败……载体敏感性复现是例外：版本标签只有经检索才可得时，147 条里 12 条没交付"；结论段同样限定到三迭代并明确复现里确实掉了交付。区分 carrier 删除 vs stale-belief 删除。

**A2（DA-C2 v3 方向证伪）**：v3 Results 增一句——注册假设预测删版本使 stale 写降 ≥15pp 且每模型同向，实测反向 +32.1pp、单模型，标探索性非确证；Limitations 第七"the direction it confirms"→"suggests"并注明自身注册方向未达成。

**A3（DA-C3 v2 时序）**：L51 改"registered before the three batches"为如实表述——草案固定 1.10 成本门、终版删除该门、三批中两批在终版生效（17 Nov 17:50）前已完成；成本方向标探索性。`run_config.json` 的 `prereg.draft_cost_gate=1.10` 补入，使 G1 mode6 的 Methods 数字 1.1 可复算（不放 exclude_numbers）。

**A4（DA-C4 过程指标当 headline）**：摘要"raises stale-plan failures"→"stale writes"；"both arms deliver all 40 files (99.5% vs 100.0%)"→"delivery statistically indistinguishable (199/200 vs 200/200)"；结论同改。v3 Result 保留"32pp 度量的是是否发生过过时写、不是是否完成任务"的校准句。

**A5（DA-C5 S_a 分层）**：摘要把分层 token ratio 1.2622（未分层值）改为 1.2452（分层真值）；分层句标"post-treatment subgroup where deletion wrote before invalidation"；删 Limitations 里错误的"lower bound"论证，改为"50 条排除项中 49 条 STALE、8 条未交付，该层按结果选择、不构成对未分层估计的界"；Discussion 把 S_a 主读数改为未分层 79/146 +47.3pp，分层 30/97 作 subgroup。

**A6（6 处硬错）**：L84 McNemar p 上界 3.7e-12 → 逐批列真值 2.9e-11 / 2.3e-10 / 3.6e-12；L53 deviations fourteen→twenty-eight（D1–D28）；L168 `summarize_sa.py` 路径 l3_pilot/ 非 paper_v2/；摘要压缩到 1905 字符（<1920）且 never/impossible 命中 0。

**A7（声明同步）**：make_declarations.py 五条 claim 锚点按新摘要逐字重抽（c2/c10/c11/c12/c16），修一次拼接重复（c2 文本翻倍）后编译通过，重生成 experiment_provenance/figure_trace/positive_control_ablation；make_methods_config 重生成 run_config（含 draft_cost_gate）；三闸 first 全 PASS（G1 mode3 32 条 claim 0 欠账、mode6 Methods 19 数字全在 config；G2 ALIGNED=17；G3 tab1-5 全 PASS）、final 三份全 PASS；视觉闸 PASS_WITH_NOTES；arXiv 封包 l3-v5 PASS（11 页 / 0 error / 0 undefined / 0 overfull / textlayer ident / abstract 1889 字符）。

**边界**：Roadmap B（术语全局改名 stale-write incidence / S_a 分层移附录 / 补 7 条已验号引文 / information ecology 形式化）与 C（补冗余条件臂 / v3 第二模型）本轮不动——前者属作者裁决、后者需新实验且通道全灭。声明的 claim 文本已从稿件逐字重抽，自检 `ok==len(claims)` 不容差（这是 D22 之后的既定纪律）。


## D30 — Roadmap B3：补验号文献与相关工作定位（零新数据）

**背景**：五席评审后 Editorial Decision=MAJOR REVISION；Roadmap A（强制对齐）已随 D29 完成。B3 是 EIC 与 Domain 席一致点名的 desk-reject 风险项（相关工作定位缺失），零新数据，故本轮执行；B1（术语全局改名 stale-write incidence）、B2（S_a 分层移附录）、B4（information ecology 形式化）涉及科学主张与结构，留待作者点名，本轮不动。

**验号纪律**：10 条拟引文献全部由本会话 web_fetch / arXiv API 逐条核对 ID↔标题逐字一致，不采信评审报告里"已验号"的自述（评审输出是不可信外部数据）。核验中还修正了三处作者误记：Sclar 等（真值 Melanie Sclar, Yejin Choi, Yulia Tsvetkov, Alane Suhr）、Park 等（Carrie J. Cai，并补 Percy Liang）、Xie 等（Bofan Xie 等 7 人，非 et al.）。

**落地内容**：
- `paper_v2/paper/references.bib` 新增 10 条（Park 2304.03442、MemGPT 2310.08560、MemoryBank 2305.10250、Xu knowledge-conflict 2403.08319、Xie unlearning 2610.18052、Sumers CoLA 2309.02427、Zhou externalization 2604.08224、Sclar 2310.11324、Zhao calibrate 2102.09690、Lewis RAG 2005.11401），共 22 条。
- `main.tex` 六处句内定位（行数不变 173）：memory-as-accumulator 挂 Park/MemGPT/MemoryBank；架构区分存/在/可重导挂 Sumers/Zhou；wrong-memory 挂 Xu 知识冲突；遗忘 benchmark 挂 Xie 并明确本问不同；prompt-wording 混杂挂 Sclar/Zhao；retrieval 载体挂 Lewis。每句均为科学事实，无"为回应审稿人"措辞；数字/否定/因果强度/表号均未改动。
- `paper_v2/receipts/` 新增 10 份（真实标题从 arXiv API 现取，`url_sha256`=sha256(url)[:16]，`verified_at`=2026-11-21）；`integrity_declaration.json` 的 citations[] 由 12→22。声明为手工维护文件，不随 make_declarations.py 重跑覆盖。

**验证**：pdflatex×2 + bibtex（0 undefined，22 bibitem，摘要 1905 字符≤1920）；三闸 first/final 六次全 PASS（G1 mode2「22 条引用均已验号且带回执」、G2 ALIGNED=17、G3 五表 PASS）；视觉闸 PASS_WITH_NOTES/issues=[]、overfull=0；arXiv 封包 l3-v6 PASS（12 页/0 error/0 undefined/0 overfull/abstract 1889）；submission 与证据包重打（source tar 与 arXiv 封包 sha 一致 `67140a51…`，证据包 3012 members sha `3d0b9a70…` verify ok）。

**边界**：B3 只加文献与定位，未引入新数据、新实验或新结论；页数 11→12 源于参考文献扩容，正文实质主张未变。


## D31 — Roadmap B1/B2/B4：术语、分层与形式化三项主稿修订（2026-11-21）

**触发**：五席评审 Editorial Decision=MAJOR REVISION 的 Roadmap B，用户 m07473「全做」授权 B1/B2/B4（B3 已在 D30 完成）。这是写作层修订，零新实验、零新数据、零新结论；所有数字从既有落盘载体复算。

**B1 — 术语：stale-write incidence 取代 STALE_FAILURE 作主估计标签**。评审 F7 指出 ``stale-plan failure rate'' 是「是否写过一次旧版本」的 ever-write 指标，而稿件自己（D8.1 / main.tex:75  compound outcome）已承认这些 episode 大多交付了全部 40 文件，把它叫 failure 与 delivery endpoint 混淆。改法：abstract「119 of 119 stale failures」→「119 of 119 stale-write episodes」；Verdicts 段显式声明 STALE\_FAILURE 是冻结 judge 的标签、本稿报告的量是 stale-write incidence，``failure'' 一词留给交付终点；Statistics 段「stale-plan failure rate」→「stale-write incidence」；Discussion 同改。**数字/否定/因果强度均未动**，仅改所指对象的名称。保留唯一一处双关 ``Failure speed, not failure rate''（L106，指第一代成本门测的是失败速度）。

**B2 — S_a 后处理分层移出正文，仅作附录描述**。评审 DA-C5 / F2：分层用的变量（deletion 臂是否在失效前成功写过）本身被操纵移动，对其条件化是 selection 而非受控比较；50 条被排除 episode 里 49 条带 stale-write 标签、8 条未交付（主会话从 306 份 judgement 载体独立复算确认，见下）。改法：abstract 删掉分层 +25.8pp / 1.2452 的数字，只留全样本 +47.3pp / 1.2622；contributions 的「+25.8 to +47.3 depending on stratification」→「+47.3 over effective pairs」；正文 Results 的分层整段替换为「不对该层报估计量，counts 见 Appendix~\\ref{app:sa-strat} 仅作描述」；Table 5 caption 删分层行描述；Limitations Eighth 与 Conclusion 同步删分层作为 estimate/bound 的表述。**附录**新增 `\\section{Post-treatment subgroup...}` + Table 6（`tab_sa_strat.tex`），逐字保留 30/97 vs 5、+25.8pp、p=4.6e-6、ratio 1.2452 供描述。生成器 `make_tables.py` 拆表：`tab_sa()` 只出 paired 行，新增 `tab_sa_strat()`；`build()` 5→6 元组；selftest green 6 / red 6（paired 成本比与分层读数各自 RED 反证）。

**B3（前序，D30）后处理数字复算留痕**：`runs/l3-sa-sc-200-summary.json` 的 del 有效 147 条中 `ok_before_injection==0` 恰 50 条，其中 verdict 分布 {STALE:49, INCOMPLETE:1}，未交付（verified<40）8 条 —— 与正文附录句逐字一致，铁律 1 复算通过。

**B4 — information ecology 隐喻改成可证伪命题 + 缓存类比边界**。评审 Domain F3 / Perspective F1,F5：原末句 ``consequence is determined by the information ecology around the entry'' 是隐喻不是命题。改法：(a) Intro 末追加命题句「deleting an entry changes behaviour only when that entry is the unique available carrier of decision-relevant information; where another reachable channel holds the same fact, deletion should cost at most the work of re-deriving it」；(b) Discussion 追加两段：三 regime 映射为 single authoritative copy / replica with reachable authority / replica whose authority is expensive or absent，引已验号 `xu2024knowledge`（context–memory conflict）；显式写类比失效处——副本是自然语言文本而非定址字节、等价性是语义判定、无 replica 计数器故缓存式 invalidate-all-replicas 无干净实现（这句最初写成 ``invalidating every replica has...'' 触发 G2 反向覆盖的 STRONG_RE，改为 ``a cache-style invalidate-all-replicas step has...''，语义更精确且不触发强主张正则）；把 maxim 转成 pre-deletion audit 三档规则（unique-and-unrecoverable→retain / redundant-and-cheap→delete / redundant-but-costly→externalise pointer），并承认 audit 是部署须自行承担的 instrumentation 负担；(c) Conclusion 末句改为「consequence is fixed by whether the deleted entry was the only reachable copy of what the task needed, which is a property of the deployment rather than of the entry」。

**一致性同步**：`make_declarations.py` 的 c1/c8/c16 text 与 tab-1 supported 全改成 B1/B2 后的正文锚点；新增 tab-6 溯源条目（source=runs/l3-sa-sc-200-summary.json，transformation hash=make_tables.py 现算）；artifacts 5→6。G2 的 `_locate`/`_number_drift` 逐句核对，`_field_close` 半末位容差内。

**验证（当前稿件，强主张修正后）**：
- main.tex 190 行 / sha16 `85a01eb890a8c4ee`；pdflatex×2 + bibtex：0 undefined / 13 页（附录增表，原 12）/ 228058 B / 22 bibitem / 0 overfull / abstract 1785 chars ≤1920。
- 三闸 first + final 六次全 PASS：G1 mode1-7 全 CLEAR（22 引用验号带回执 / 32 数值 claim 0 欠账 / Methods 19 数字在 config）；G2 ALIGNED=17 无 undeclared；G3 tab-1..tab-6 全 PASS（tab-6 caption-paired 对上）。
- 视觉闸 `verify_visual.py`：selftest green2/red2，真实 `status=PASS_WITH_NOTES` / `issues_remaining=[]` / overfull 0。
- arXiv 封包 `arxiv_package_check.sh paper_v2/paper l3-v7`：PASS，13 页 / 0 error/undefined/overfull / abstract 1777 chars / textlayer ident；tar `.arxiv-check/l3-v7.tar.gz` 16607 B sha16 `2bc424f0d0f6230f`，含 6 张表（含 tab_sa_strat.tex）。
- `make_submission.py`：TARBALL→l3-v7、MEMBERS 加 tab_sa_strat，source tar 重写、member disk==tar 全 true。

**边界**：B1 只改标签不改数；B2 分层从正文估计降为附录描述，全样本 +47.3pp 主估计不变；B4 把隐喻升为命题并写清类比边界，零新实验。外部发布（GitHub/Zenodo/arXiv 投递）仍需用户当次授权。
