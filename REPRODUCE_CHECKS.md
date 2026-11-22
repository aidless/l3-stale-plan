# L3 复核命令清单（每次出结论前跑一遍，全绿才准写数）

```bash
# 1. 冻结锚点（PREREG §1 哈希 + 功效参数）
python3 l3_pilot/power_l3.py --check                     # 期望 exit 0

# 2. 判定口径围栏（冻结 6 文件哈希 + 统计口径）
python3 l3_pilot/formal_runner.py --verify-provenance    # 期望 exit 0, frozen_ok: true

# 3. 证据链（report/judgements/traces/state 逐条对齐；聚合表无载体 exit 2）
python3 l3_pilot/verify_evidence_chain.py --selftest     # 期望 4/4 PASS
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-runA-50x2x3  # 期望 exit 0, n_records=300

# 4. 聚合表若被引用，必须是 exit 2（无载体 → INFERRED）
python3 l3_pilot/verify_evidence_chain.py --aggregate l3_pilot/runs/l3-fact-240x6x2-analysis/factorial_2880.json --shards-glob 'l3_pilot/runs/l3-fact-240x6x2-shard*/judgements' ; echo "expect_exit=2"
```

## v2 S_c 主实验（D12，2026-11-17）

```bash
python3 l3_pilot/v2_runner_sc.py --selftest                     # 期望 SELFTEST PASS (8/8，含第五类 PIC)
python3 -c "import json; r=json.load(open('l3_pilot/runs/l3-v2-sc-40-minimax/report.json')); print(r['cells']['keep']['n'], r['cells']['del']['n'])"
# 期望: keep 40 / del 40；judgements 80、traces 80、transport_loss 0
# 配对: n_effective=36, keep_stale 0/36, del_stale 36/36, p=2.91e-11, cost 3.21x
```
- 阳性对照门（D11.5）：`runs/l3-v2-pc20-minimax/` keep 17/20 REC + del 20/20 STALE；`runs/l3-v2-pc20-workbuddy/` 剔 429 后 keep 13/13 REC + del 9/10 STALE —— 两模型过线后才开的 40 卡主实验。

### v2 S_c runs（2026-11-17）
```bash
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-v2-sc-40-minimax  # 期望 exit 0, n=80（D12 主确证）
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-v2-sc-40-kimi     # 期望 exit 0, n=80（65/80 transport_loss，方向性佐证）
python3 l3_pilot/verify_evidence_chain.py --run-dir l3_pilot/runs/l3-v2-sc-40-kimi-lowspeed  # 期望 exit 0, n=80, TL=3/80（D12.5 第二确证：del 40/40 STALE，n_eff=33，p=2.33e-10，cost 3.76）
```
注：闸已适配 v2 runner（state.json 只存 done_slugs、records=judgements/ 逐份对齐）；篡改 n_records 副本实测仍红。

### transport_loss 清污闸（D12.4，低速补跑用）
```bash
python3 l3_pilot/purge_v2_transport_loss.py --selftest   # GREEN 只清 TL、RED 按 TL 不按 verdict，2 PASS
python3 l3_pilot/purge_v2_transport_loss.py --run-dir l3_pilot/runs/l3-v2-sc-40-kimi-lowspeed           # 只报告，不改盘
python3 l3_pilot/purge_v2_transport_loss.py --run-dir l3_pilot/runs/l3-v2-sc-40-kimi-lowspeed --apply    # 清 TL + 同名 trace + done_slug
```
判定：真 INCOMPLETE / STALE / RECOVERED 一律保留；`transport_loss=True` 的格子才能重试。低速批监督器 `/tmp/kimi_lowsup.sh` 每轮 purge→preflight 3×200→`--resume --workers 1`。

### 第五类判定与五分重判层（D13，2026-11-17 v2 预注册生效）
```bash
python3 l3_pilot/v2_runner_sc.py --selftest              # 8/8（PIC 命中 / 截断优先 / 假阳不误判）
python3 l3_pilot/rejudge_v2_verdicts.py --selftest       # 2/2（GREEN PIC 单列 + RED verified<n_files 不判 PIC）
python3 l3_pilot/rejudge_v2_verdicts.py                  # 期望：PIC 3(keep)/4(keep)，STALE 40/40、REC 36/34 逐条不变
```
判定：`PRE_INJECTION_COMPLETION` 不进 stale 分母、不进完成率分子；`truncated→INCOMPLETE_TRUNCATED` 优先于 PIC。
旧 judgements/report **不改**（保跨批可比），五分口径只在 `runs/l3-v2-rejudged/` 并列存在。

## D14 段：第三模型 workbuddy 零折损批（2026-11-17 19:18 PST）

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot

# 证据链：期望 ok=True n=80
python3 verify_evidence_chain.py --run-dir runs/l3-v2-sc-40-workbuddy2

# 关键读数：期望 40 97.5 3.637978807091713e-12 3.6553
python3 -c "
import json; r=json.load(open('runs/l3-v2-sc-40-workbuddy2/report.json'))
p=r['paired']; print(p['n_effective'], p['diff_pp'], p['mcnemar_exact_p'], p['cost_ratio_del_over_keep'])"

# 五分重判（三批一起；workbuddy 期望 mismatch={} 即四分=五分）
python3 rejudge_v2_verdicts.py --runs l3-v2-sc-40-minimax l3-v2-sc-40-kimi-lowspeed l3-v2-sc-40-workbuddy2
```

三模型合并（同一 S_c 载体、同一卡集、同一判定）：

| 模型 | TL | n_eff | diff | McNemar p | cost ratio |
|---|---|---|---|---|---|
| minimax-m3.1-flash | 0 | 36 | 100pp | 2.91e-11 | 3.21× |
| **workbuddy/deepseek-v4.1-flash** | **0** | **40** | **97.5pp** | **3.64e-12** | **3.66×** |
| kimi-k2-thinking（低速批） | 3/80 | 33 | 100pp | 2.33e-10 | 3.76× |

有效配对合计 **109**，仍远低于 §6 的 420 → 结论停在「方向 + 效应量级」，成本比不得写成已精确估计。

## D15 视觉合规（figure_verification_skill 十条 checklist 的本地可执行版）

modlens 六个 provider 全未配置（`npx @liustack/modlens doctor` 实测全 `[!!]`），按 skill §1
整条流程不可用，必须显式报 SKIPPED 而不是假装通过。本稿无图、只有两张 booktabs 表，
客观判据用 pymupdf span 级读数确定性判掉：

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 paper_v2/scripts/verify_visual.py --selftest   # 期望 SELFTEST PASS (green 2 / red 2)
python3 paper_v2/scripts/verify_visual.py              # 期望 status=PASS_WITH_NOTES, rc=0
```

判据来源（全部现读，无手抄）：字号 `page.get_text("dict")` 的 `span["size"]`；版心
`page.rect.width - 2*1in`；表内数值对 `paper_v2/evidence/batch_counts.json` 与
`paper_v2/stats.json` 逐格对账。

实测读数：页面 612x792pt（letter）、7 页、版心 468.0pt、3245 个 span、<8pt 的 12 个
全是数学指数上标（6.97/7.97pt，字体 CMR7/CMR8/CMSY7/CMSY8，共 18 字符）；
最宽表规则线 428.7pt（slack 39.3pt）；21 个矢量对象、填充色仅黑白；pdflatex
overfull/underfull 均 0。逐格对账 33 项全等。

RED 反证：`check_07_sizes` 的 RED-1 首版未触发——`is_math_script` 靠字体名判上标，
而 `\fontsize{7}{8}\selectfont` 造的 7pt **正文**也用 CMR7，字体名读不出「是不是上标」。
改成三条独立判据（字号 <8pt / 文本≤3 字符且只含数字与运算符 / origin y 真抬高）后 RED 才红。
```

## D16 — 零成功写诊断（2026-11-18）

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
# 独立判定层（不碰注册重判载体）
python3 d16_zero_write_layer.py --selftest        # SELFTEST PASS (3/3)
python3 d16_zero_write_layer.py --runs l3-v2-sc-40-minimax l3-v2-sc-40-kimi-lowspeed l3-v2-sc-40-workbuddy2 --out runs/l3-v2-d16
# 期望：minimax/workbuddy 各 STALE_FAILURE->NO_VALID_WRITE:1，kimi moved={}
python3 paper_v2/scripts/measure_writes.py        # 重写 success_writes.json，selftest 2/2
python3 paper_v2/scripts/make_tables.py           # 三张表；tab_writes 行尾必须为双反斜杠
cd paper_v2/paper && pdflatex -interaction=nonstopmode main.tex && pdflatex -interaction=nonstopmode main.tex
cd .. && python3 paper_v2/scripts/make_declarations.py   # 三表 caption 全入 figure_trace
python3 "/home/liuzewen/F:\deepseek/scripts/research_failure_mode_gate.py" --decl paper_v2/integrity_declaration.json --paper paper_v2/paper/main.tex --root .
python3 "/home/liuzewen/F:\deepseek/scripts/experiment_provenance_gate.py" --decl paper_v2/experiment_provenance.json --paper paper_v2/paper/main.tex --root .
python3 "/home/liuzewen/F:\deepseek/scripts/figure_trace_gate.py" --manifest paper_v2/figure_trace.json --paper paper_v2/paper/main.tex --root .
python3 paper_v2/scripts/verify_visual.py          # PASS_WITH_NOTES, issues_remaining=[]
```

关键读数：del 三批各 40/40 `never_ok`、注入前成功写 0/224、0/334、0/104；
keep 用对资源名 39/40、39/40、40/40；`NO_VALID_WRITE` 只在 minimax/workbuddy 各 1 条；
kimi 因 271 次 stale 尝试一条不动。

## D17：report 与载体对账（第四模型批 + 两处漏账修复）

```bash
cd l3_pilot
# 1) 自检：3 GREEN（三批已知批次重算）+ 2 RED（删载体必须红 / 空 judgements 必须拒）
python3 resummarize_v2.py --selftest
#    期望：SELFTEST PASS (5/5)

# 2) 缺 report 的批次（只读模式先看 drift）
python3 resummarize_v2.py --run-dir runs/l3-v2-sc-40-hy4free2
#    期望：carrier_judgements=29、drift_vs_old_report 为空（该批原本没有 report）

# 3) 已登记批次：只查 drift，禁止 --apply（report 已被 evidence_manifest 按 sha256 钉住）
python3 resummarize_v2.py --run-dir runs/l3-v2-sc-40-kimi-lowspeed
#    期望 drift 恰好两项：paired.cost_ratio_del_over_keep 3.7622→3.8151、cells.del.median_tokens 35960→36859

# 4) 证据链
python3 verify_evidence_chain.py --run-dir runs/l3-v2-sc-40-hy4free2
#    期望：{"ok": true, "n_records": 29, "errors": []}
```

第四模型 `workbuddy/hy4flash-free`（`runs/l3-v2-sc-40-hy4free2`）：keep 17 条 REC 11（0.647，**未达 0.8 阳性对照门**）、del 12 条全 STALE；配对 n_effective=10、diff 100.0pp、McNemar p=0.001953125、cost_ratio 2.0517。
按 `L3_V2_PREREG_REVIEWABLE.md` §4.1 四档，该批**只作方向性佐证**，不进 Table 1/2，不进 pooled 109 配对。

## D18（2026-11-19）v3 隔离设计与 arXiv 定稿闸

```bash
cd l3_pilot
python3 v3_runner.py --selftest                     # 期望 SELFTEST PASS (6/6)
python3 v3_smoke.py --cards 2 --out runs/l3-v3-smoke # smoke：del 注入前成功写 >0（非 v2 零写）
cd paper_v2
python3 scripts/make_tables.py                       # 写 paper/tables/tab_{cells,paired,writes}.tex
bash "/home/liuzewen/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh" paper l3-source
# 期望 status PASS / 0 error / 0 undefined / 0 overfull / textlayer cmp ident / 9 pages
```
三闸与视觉闸按既有命令重跑，first/final 均 exit 0；`verify_visual.py` 期望 PASS_WITH_NOTES。

## D19 — v3 阳性对照与方向证伪（2026-11-19）

```bash
cd l3_pilot
# 预注册签署态
head -5 L3_V3_PREREG.md                    # APPROVED / IN FORCE 2026-11-19
grep 1d92fbcd L3_V3_PREREG.md              # v3_runner 冻结身份

# runner 自检
python3 v3_runner.py --selftest             # SELFTEST PASS 7/7

# 阳性对照读数（已固化，runner 原生 report + 独立汇总）
cat runs/l3-v3-pc40-minimax/positive_control.json
#   期望: n_records=80 TL=1; keep REC 37/STALE 0/PIC 2; del REC 26/STALE 8/PIC 6
#   配对 n_effective=31, keep_stale=0 del_stale=8, diff_pp=25.8065,
#   mcnemar_exact_p=0.0078125, cost_ratio=0.9457
cat runs/l3-v3-pc40-minimax/report.json
#   writes 块: keep episodes_with_ok_write=39, del=40 (D16 混杂解除)

# 证据链
python3 verify_evidence_chain.py --run-dir runs/l3-v3-pc40-minimax   # ok=True n=80
```
判据：阳性对照门 keep 有效 REC 率 1.0≥0.8、del 退化 25.81pp≥0.5pp 全过；
但 H3v3「删版本映射降 STALE≥15pp」方向**证伪**（实测删了反而 +25.81pp），见 D19。

## D20 — v3 主实验 200 卡批（2026-11-19）

```bash
cd l3_pilot
python3 summarize_v3_main.py                            # 期望: n_records=400 TL=0
#   keep n=200 REC174/STALE2/INC1/PIC23; del n=200 REC123/STALE59/PIC18
#   配对 n_valid=200 n_effective=159; improved=1 worsened=52;
#   diff_pp=32.0755; mcnemar_exact_p=1.199040866595169e-14; cost_ratio=1.0177
python3 summarize_v3_main.py runs/l3-v3-pc40-minimax    # 对表 40 卡: 31 对 25.8065pp p=0.0078125
python3 verify_evidence_chain.py --run-dir runs/l3-v3-sc-200-minimax   # ok=True n=400
```
判据：159 有效对 ≥140 门槛（power_ok）；方向 deletion_harmful，McNemar 极显著；
成本比 1.0177≤1.10。H3v3 方向证伪，见 D19/D20。

## D20.1 — 交付完成率第二读数

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
# 期望: keep 200 199 / del 200 200
```
判据：两臂交付完成率无实质差异（99.5% vs 100.0%）；del 59 条 STALE_FAILURE 的 verified 全部=40。
因此 D20 的 32.08pp 度量「是否产生过基于过时版本的写」，不是「任务是否做成」。

## D21 — 三层链条叙事入稿（v3 进稿）

```bash
cd l3_pilot
# v3 数字现算（表4与声明的唯一来源）
python3 paper_v2/scripts/make_v3_evidence.py --selftest   # 期望 SELFTEST PASS (green 3 / red 1)
python3 paper_v2/scripts/make_v3_evidence.py >/dev/null   # 产物 paper_v2/evidence/v3_main.json
# 四张表
python3 paper_v2/scripts/make_tables.py --selftest        # 期望 SELFTEST PASS (green 4 / red 4)
python3 paper_v2/scripts/make_tables.py >/dev/null        # tables/tab_{cells,paired,writes,v3}.tex
# 声明（claim 逐字抽、tab-4 trace）
python3 paper_v2/scripts/make_declarations.py --selftest  # 期望 SELFTEST PASS (green 15 / red 1)
python3 paper_v2/scripts/make_declarations.py             # experiment_provenance/figure_trace/positive_control_ablation
# 编译
cd paper_v2/paper && pdflatex -interaction=nonstopmode main.tex >/dev/null 2>&1 && pdflatex -interaction=nonstopmode main.tex >/dev/null 2>&1; cd ../..
# 三闸 first
G="/home/liuzewen/F:\deepseek/scripts"
python3 "$G/research_failure_mode_gate.py" --decl paper_v2/integrity_declaration.json --paper paper_v2/paper/main.tex --root . --json-out paper_v2/reports/gate1_first.json
python3 "$G/experiment_provenance_gate.py" --decl paper_v2/experiment_provenance.json --paper paper_v2/paper/main.tex --root . --json-out paper_v2/reports/gate2_first.json
python3 "$G/figure_trace_gate.py" --manifest paper_v2/figure_trace.json --paper paper_v2/paper/main.tex --root . --json-out paper_v2/reports/gate3_first.json
# 三闸 final（各加 --final --prior paper_v2/reports/gateN_first.json）
# 视觉闸
python3 paper_v2/scripts/verify_visual.py                  # 期望 status=PASS_WITH_NOTES, issues_remaining=[]
# arXiv 封包
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper_v2/paper l3-v3  # 期望 PASS, 10 pages, abstract 1570 chars
```
期望读数：v3 159 有效对 / diff 32.0755pp / p=1.199e-14 / cost 1.0177 / del 注入前写 1481 / 交付 199 vs 200 / del STALE 59 条全搬完 40 文件。G1 mode3=17 条 claim 当场核对、G2 ALIGNED=13、G3 tab1-4 全 PASS。冻结层 6 文件哈希不变。

### D22 段：投稿包与证据包（2026-11-20 PST）

```bash
cd l3_pilot

# 1) 投稿包（tarball 逐文件对表磁盘稿件，不一致即 exit≠0）
python3 paper_v2/scripts/make_submission.py --selftest
# 期望：SELFTEST PASS (green 1 / red 2)
python3 paper_v2/scripts/make_submission.py
# 期望：写 paper_v2/submission/submission_manifest.json，members 六项 match=true

# 2) 证据包（载体计数必须与 report.json.n_records 一致）
python3 paper_v2/scripts/make_release_pack.py --selftest
# 期望：SELFTEST PASS (green 1 / red 2)
python3 paper_v2/scripts/make_release_pack.py
# 期望：l3-evidence-<date>.tar.gz，2223 文件
python3 paper_v2/scripts/make_release_pack.py --verify paper_v2/submission/l3-evidence-*.tar.gz
# 期望：{"files": 2223, "ok": true}
(cd paper_v2/submission && sha256sum -c SHA256SUMS | grep -c OK)
# 期望：2223

# 3) Zenodo 上传脚本（默认只建草稿，--publish 才铸 DOI；selftest 用本机桩）
python3 paper_v2/scripts/zenodo_deposit.py --selftest
# 期望：SELFTEST PASS (green 2 / red 1)
```

实际发布（`gh release create` / Zenodo 上传）**未执行**：按 AGENTS.md §3，外部发布需用户当次明确授权。前置条件已实探：`gh` 已登录 `aidless`；`ZENODO_TOKEN` 环境为空。命令模板见 `paper_v2/submission/README_RELEASE.md`。

## D24 — S_a 敏感性臂（检索语料载体）

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 census_sa.py --selftest        # 期望 SELFTEST PASS（普查谓词自证）
python3 sa_runner.py --selftest        # 期望 SELFTEST PASS (8/8)
python3 summarize_sa.py --selftest     # 期望 SELFTEST PASS (2/2)，先对表 2 卡 smoke 8 项
python3 summarize_sa.py --run-dir runs/l3-sa-sc-40
#   期望 n_records 80 / TL 0 / keep REC 35 STALE 5 / del REC 15 STALE 25
#   paired diff_pp 50.0 / mcnemar 8.797645568847656e-05 / cost_ratio 1.3341
#   writes: keep delivered_all 40 ok_before 61；del delivered_all 37 ok_before 44
python3 verify_evidence_chain.py --run-dir runs/l3-sa-sc-40   # 期望 ok=true n_records=80
cat runs/l3-sa-sc-40-stratified.json
#   期望 stratum del_pre_ok_gt_0 / n_pairs 25 / diff_pp 28.0 / p 0.09228515625 / cost 1.256
```

**引用纪律**：S_a 全样本 50pp 混入「注入前只检索不写」行为差异（del 15/40 条 zero_pre，
keep 0/40），只能作方向性佐证；分层后 28pp 方向一致但 n=25 不显著。交付率 keep 40/40、
del 37/40 必须与 stale 率并列报告（D20.1）。

## D25 — S_a 扩样读数（306 条，通道耗尽提前收尾）

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot
python3 summarize_sa.py --selftest                       # 期望 SELFTEST PASS (4/4)
python3 resummarize_sa.py --selftest                     # 期望 SELFTEST PASS (2/2)，40 卡 report 逐字段对表
python3 summarize_sa.py --run-dir runs/l3-sa-sc-200 --out runs/l3-sa-sc-200-summary.json
#   期望 n_records=306 / TL=0 / keep REC137 STALE10 / del REC67 STALE79
#   paired n_eff=146 keep_stale 0.0685 del_stale 0.5411 diff_pp 47.2603
#   mcnemar p=3.72567441849804e-18 / cost_ratio 1.2622
#   writes keep ok_before 285 delivered 147/147；del ok_before 148 delivered 135/147
#   stratified n_pairs=97 keep_stale 5 del_stale 30 diff_pp 25.7732 p=4.649162292480469e-06 cost 1.2452
python3 verify_evidence_chain.py --run-dir runs/l3-sa-sc-200   # 期望 ok=true n_records=306
```

**读法**：全样本 47.26pp 含 del 臂 50 张注入前零成功写的混杂；分层（del 注入前写过，n=97）
25.77pp p=4.65e-06 显著、方向稳住。交付率必须并列：keep 147/147=100%、del 135/147=91.84%
（del 79 条 STALE 里 68 条仍搬完 40 文件）。分层方向标签以 `summarize_sa.py::stratified()`
为准（improved=3/worsened=10 @40卡、3/28 @扩样），旧 `l3-sa-sc-40-stratified.json`
标 SUPERSEDED。S_a 单模型、成本 1.26>1.10、stop-at-306，仅作机制一致性佐证，不改主稿。

## D26 — S_a 入稿（四层链条）复核命令

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot

# 表生成器（含 Table 5）：selftest green5/red5，然后生成五张表
python3 paper_v2/scripts/make_tables.py --selftest          # 期望 SELFTEST PASS (green 5 / red 5)
python3 paper_v2/scripts/make_tables.py                     # 写 tables/tab_cells|paired|writes|v3|sa.tex

# S_a 载体现算（新尺验底物：40 卡对表在 selftest 里）
python3 summarize_sa.py --selftest                          # 期望 SELFTEST PASS (4/4)
python3 summarize_sa.py --run-dir runs/l3-sa-sc-200          # 期望 306 条 / diff 47.2603 / p 3.73e-18 / strat n97 diff 25.77 p 4.65e-06
python3 resummarize_sa.py --selftest                        # 期望 SELFTEST PASS (2/2，40 卡对表)

# 方法学 config + 三闸声明（从载体现算，不手抄）
python3 paper_v2/scripts/make_methods_config.py --selftest  # 期望 SELFTEST PASS (4/1)
python3 paper_v2/scripts/make_declarations.py --selftest    # 期望 SELFTEST PASS (green 19 / red 1)
python3 paper_v2/scripts/make_methods_config.py
python3 paper_v2/scripts/make_declarations.py                # tables hash sha256:8b14b4f794b8a2a5

# 编译（tables/ 子目录，arXiv 封包白名单内）
cd paper_v2/paper && pdflatex -interaction=nonstopmode main.tex && pdflatex -interaction=nonstopmode main.tex
# 期望 errs=0 / overfull=0 / 11 pages

# ARS 三闸（权威路径，--root .；cd 回 l3_pilot）
cd ../..
G="/home/liuzewen/F:\\deepseek/scripts"
python3 "$G/research_failure_mode_gate.py" --decl paper_v2/integrity_declaration.json --paper paper_v2/paper/main.tex --root .   # 期望 PASS，mode3 32 条 claim 全字段核对，0 欠账
python3 "$G/experiment_provenance_gate.py" --decl paper_v2/experiment_provenance.json --paper paper_v2/paper/main.tex --root .   # 期望 PASS，ALIGNED=17
python3 "$G/figure_trace_gate.py" --manifest paper_v2/figure_trace.json --paper paper_v2/paper/main.tex --root .                 # 期望 PASS，tab-1..tab-5
# 终闸复查（first 报告在 paper_v2/reports/gate{1,2,3}_first.json）
python3 "$G/research_failure_mode_gate.py" --decl paper_v2/integrity_declaration.json --paper paper_v2/paper/main.tex --root . --final --prior paper_v2/reports/gate1_first.json   # PASS
python3 "$G/experiment_provenance_gate.py" --decl paper_v2/experiment_provenance.json --paper paper_v2/paper/main.tex --root . --final --prior paper_v2/reports/gate2_first.json  # PASS
python3 "$G/figure_trace_gate.py" --manifest paper_v2/figure_trace.json --paper paper_v2/paper/main.tex --root . --final --prior paper_v2/reports/gate3_first.json             # PASS

# 视觉闸 + arXiv 封包
python3 paper_v2/scripts/verify_visual.py --selftest        # 期望 green2/red2
python3 paper_v2/scripts/verify_visual.py paper_v2/paper/main.pdf   # 期望 status=PASS_WITH_NOTES, issues_remaining=[]
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper_v2/paper l3-v4
# 期望 PASS：11 页 / 0 error / 0 undefined / 0 overfull / textlayer ident / abstract 1867 chars < 1920
```

关键读数（Table 5）：keep 147(137 REC/10 STALE) delivered 147/147 ok-before 285 med 36,401；del 147(67 REC/79 STALE) delivered 135/147 ok-before 148 med 50,079；paired n146 79 vs 10 diff 47.3pp p 3.73e-18 cost 1.2622；stratified n97 30 vs 5 diff 25.8pp p 4.65e-06 cost 1.2452。定位为单模型敏感性佐证，非第四次预注册迭代。

## D27 — 投稿包与证据包复核命令

```bash
cd /home/liuzewen/research-plan-20261108/l3_pilot

# 三个封包生成器的自证（都会变红）
python3 paper_v2/scripts/make_submission.py --selftest     # green1/red2：篡改与截断 tar 必被拒
python3 paper_v2/scripts/make_release_pack.py --selftest   # green1/red2：损坏与缺批必被拒
python3 paper_v2/scripts/zenodo_deposit.py --selftest      # green2/red1：默认草稿、--publish 才铸 DOI

# 重建源包与投稿清单（要求 l3-v4.tar.gz 存在；六成员逐字节一致）
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper_v2/paper l3-v4
python3 paper_v2/scripts/make_submission.py                # 复制 l3-v4 -> submission/l3-source.tar.gz

# 重建证据包（要求每批 judgement/trace 数与 report.n_records 相等）
python3 paper_v2/scripts/make_release_pack.py              # 期望 3011 文件 / 14 批
python3 paper_v2/scripts/make_release_pack.py --verify paper_v2/submission/l3-evidence-20261121.tar.gz
# 期望 {"files": 3012, "ok": true}

# 解包后逐文件校验（关键：路径相对包根，必须解包后再跑）
rm -rf /tmp/l3v && mkdir /tmp/l3v
tar xzf paper_v2/submission/l3-evidence-20261121.tar.gz -C /tmp/l3v
cd /tmp/l3v && sha256sum -c /home/liuzewen/research-plan-20261108/l3_pilot/paper_v2/submission/SHA256SUMS
# 期望 3012 行全 OK，0 FAILED

# 治理文档完整性（上一版静默丢了 3 份，D27.1-3）
tar tzf paper_v2/submission/l3-evidence-20261121.tar.gz | grep -E 'PREREG_L3|L3_V2_PREREG_REVIEWABLE|L3_V3_PREREG|REPRODUCE_CHECKS|DEVIATION_L3'
# 期望五份都在
```

关键读数：源包 sha16 `7c49e880384152ebb890`（与 `l3-v4.tar.gz` 逐字节相同，含 tab_sa.tex）；证据包 3011 文件 / 14 批 / 未压缩 16,276,221 B；`SHA256SUMS` 3012/3012 OK。外部发布（GitHub Release / Zenodo）未执行，命令模板在 `paper_v2/submission/README_RELEASE.md`。

## D29 复核命令（Roadmap A 修订后）

稿件改动后所有声明/生成链/封包都必须重验。以下命令在 `l3_pilot/` 下执行：

```bash
# 编译 + 摘要长度
cd paper_v2/paper && pdflatex -interaction=nonstopmode main.tex && pdflatex -interaction=nonstopmode main.tex
# abstract 1905 字符 (<1920)，never/impossible 命中 0

# 生成链自检 + 重生成
python3 paper_v2/scripts/make_tables.py --selftest       # green5/red5
python3 paper_v2/scripts/make_declarations.py --selftest # green19/red1
python3 paper_v2/scripts/make_methods_config.py --selftest # green4/red1
python3 paper_v2/scripts/verify_visual.py --selftest     # green2/red2
python3 paper_v2/scripts/make_declarations.py            # 重生成三声明
python3 paper_v2/scripts/make_methods_config.py          # 重生成 run_config（draft_cost_gate=1.10）
python3 paper_v2/scripts/make_tables.py                  # 五张表

# 三闸 first + final（权威脚本在工作区根，--root .）
python3 "/home/liuzewen/F:\deepseek/scripts/research_failure_mode_gate.py" --decl paper_v2/integrity_declaration.json --paper paper_v2/paper/main.tex --root . --json-out paper_v2/reports/gate1_first.json   # PASS, mode6 Methods 19 数字全在 config
python3 "/home/liuzewen/F:\deepseek/scripts/experiment_provenance_gate.py" --decl paper_v2/experiment_provenance.json --paper paper_v2/paper/main.tex --root . --json-out paper_v2/reports/gate2_first.json  # ALIGNED=17
python3 "/home/liuzewen/F:\deepseek/scripts/figure_trace_gate.py" --manifest paper_v2/figure_trace.json --paper paper_v2/paper/main.tex --root . --json-out paper_v2/reports/gate3_first.json  # tab1-5 PASS
# 各加 --final --prior paper_v2/reports/gateN_first.json → 全 PASS

# 视觉 + arXiv 封包
python3 paper_v2/scripts/verify_visual.py               # PASS_WITH_NOTES
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper_v2/paper l3-v5  # PASS: 11 页/0 error/0 undefined/0 overfull/abstract 1889

# 三键读数（改稿未动载体，应与 D26/D27 一致）
python3 summarize_v3_main.py runs/l3-v3-sc-200-minimax   # n159 diff32.0755 p1.199e-14 ratio1.0177
python3 summarize_sa.py --run-dir runs/l3-sa-sc-200       # n146 diff47.2603 p3.726e-18 ratio1.2622; strat n97 25.7732 1.2452
```

关键期望：三闸 first/final 全 exit 0；G1 mode3 "32 条 claim 0 欠账"、mode6 "Methods 19 数字全部能在 run config 中找到"；摘要 1905 字符、never/impossible 命中 0；PDF 11 页 0 overfull。



## B3 复算命令（Roadmap B3：补验号文献与定位，2026-11-21；零新数据）

前置：全程在 `l3_pilot/` 下操作；引用验号由本会话 web_fetch/arXiv API 逐条核对 ID↔标题，不采信评审自述。

**1. 引文验号（10 条，逐条 ID↔标题逐字一致）**
- arXiv API：`https://export.arxiv.org/api/query?id_list=<comma-separated 10 ids>&max_results=10`，从 `<entry>` 抽 `<id>`/`<title>`。
- 10 条：Park 2304.03442 / MemGPT 2310.08560 / MemoryBank 2305.10250 / Xu knowledge-conflict 2403.08319 / Xie unlearning 2610.18052 / Sumers CoLA 2309.02427 / Zhou externalization 2604.08224 / Sclar 2310.11324 / Zhao calibrate 2102.09690 / Lewis RAG 2005.11401。
- 作者核验修正三处：Sclar (Melanie Sclar, Yejin Choi, Yulia Tsvetkov, Alane Suhr)、Park (补 Carrie J. Cai + Percy Liang)、Xie (Bofan Xie 等 7 人，非 et al.)。
- 期望：10 条全部 title 匹配；receipts/ 新增 10 份，`url_sha256`=sha256(url)[:16]、`verified_at`=2026-11-21；`integrity_declaration.json` citations 12→22。

**2. 编译链**
```bash
cd l3_pilot/paper_v2/paper
pdflatex -interaction=nonstopmode -halt-on-error main.tex
bibtex main
pdflatex -interaction=nonstopmode -halt-on-error main.tex
pdflatex -interaction=nonstopmode -halt-on-error main.tex
```
期望：0 undefined citations、22 bibitem、12 页、摘要 1905 字符≤1920。

**3. 三闸 first/final（权威脚本在含字面反斜杠的工作区根 scripts/）**
```bash
cd l3_pilot
python3 '/home/liuzewen/F:\deepseek/scripts/research_failure_mode_gate.py' --root . --input paper_v2/integrity_declaration.json ... --json paper_v2/reports/gate1_first.json
# 另两闸 experiment_provenance_gate.py / figure_trace_gate.py 同式；六次 first/final 全 exit 0
```
期望：G1 mode2「22 条引用均已验号且带回执」、mode3「32 条 claim 0 欠账」、mode6 19 数字；G2 ALIGNED=17；G3 五表 PASS；终闸复查三份 PASS。

**4. 视觉闸 + arXiv 封包**
```bash
python3 paper_v2/scripts/verify_visual.py     # status=PASS_WITH_NOTES, issues_remaining=[], overfull=0
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper l3-v6
```
期望：arXiv 封包 l3-v6 PASS — 12 页/0 error/0 undefined/0 overfull/abstract 1889 chars；tar sha256 `67140a5140770bf4f1ebf480…`。

**5. submission / 证据包重建与校验**
```bash
cd l3_pilot
python3 paper_v2/scripts/make_submission.py            # TARBALL=l3-v6.tar.gz；source tar sha16=67140a5140770bf4f1ebf480
python3 paper_v2/scripts/make_submission.py --selftest  # SELFTEST PASS (green 1 / red 2)
python3 paper_v2/scripts/make_release_pack.py          # 3012 members
python3 paper_v2/scripts/make_release_pack.py --verify paper_v2/submission/l3-evidence-20261121.tar.gz
```
期望：verify ok=True/files=3012/errors=[]；bundle sha256 `3d0b9a708490d0843c5431d2f070f915f8a09d471d65096a6517c35fb97881e5`；README_RELEASE.md 记录的 sha 与盘上一致；source tar 与 arXiv 封包逐字节相同。

**边界**：B3 仅加文献与定位，零新数据/新实验/新结论；页数 11→12 源于参考文献扩容；正文数字/否定/因果强度/表号未改动。D30 已同步 DEVIATION_L3.md（前缀 cfc0d0a4471c980a→977e349780a8445f，^##D=34→35）。

## B1/B2/B4 复算命令（Roadmap B1/B2/B4：术语 / 分层归位 / 机制形式化，2026-11-21；零新数据）

三项改动均为**表述与结构**改动：B1 把结局标签从 "stale-plan failure" 改为 "stale-write incidence / stale-write episodes"（failure 一词只保留给交付端点）；B2 把 S_a 的后处理分层估计从正文降为附录描述（正文只留全样本配对）；B4 把 "information ecology" 隐喻升为可证伪命题并写清缓存类比的失效边界。没有新实验、没有新数据集、没有改动任何原始载体。

**1. 表格生成（拆表：正文 5 表 + 附录 1 表）**
```bash
cd l3_pilot
python3 paper_v2/scripts/make_tables.py --selftest
# 期望 SELFTEST PASS (green 6 / red 6)  cells=14 paired=11 writes=14 v3=10 sa=10 sa_strat=7 lines
python3 paper_v2/scripts/make_tables.py
# 期望写出 6 个 tex；tab_sa.tex 只含 paired 行（146 & 79 vs 10 & ×1.2622），
# tab_sa_strat.tex 只含 post-treatment 行（97 & 30 vs 5 & +25.8 & ×1.2452）
```

**2. 声明同步 + 溯源/trace/正对照重生成**
```bash
cd l3_pilot
python3 paper_v2/scripts/make_declarations.py --selftest
# 期望 SELFTEST PASS (green 19 / red 1)
python3 paper_v2/scripts/make_declarations.py
# 期望 wrote experiment_provenance.json figure_trace.json positive_control_ablation.json
#      tables hash sha256:bb99f4bf3f5f4fce；artifacts/traces 均为 tab-1..tab-6
```
关键点：`integrity_declaration.json` 无生成器（手工维护），B1/B2 改正文句后 `make_declarations.py` 的 c1/c8/c16/tab-1/tab-6 文本必须逐字对齐正文，否则 G2 报"声明与稿件脱节"。

**3. 编译链（两遍 + bibtex）**
```bash
cd l3_pilot/paper_v2/paper
pdflatex -interaction=nonstopmode main.tex && bibtex main && pdflatex main && pdflatex main
# 期望 0 undefined / 0 overfull / 13 页 / 22 bibitem / abstract 1785 字符 (<1920)
```

**4. 三闸 first + final（权威脚本在含字面反斜杠的工作区根 scripts/，必须先 cd 到 l3_pilot）**
```bash
cd l3_pilot
G='/home/liuzewen/F:\deepseek/scripts'
python3 "$G/research_failure_mode_gate.py"   --root . --decl paper_v2/integrity_declaration.json   --paper paper_v2/paper/main.tex --json-out paper_v2/reports/gate1_first.json
python3 "$G/experiment_provenance_gate.py"   --root . --decl paper_v2/experiment_provenance.json  --paper paper_v2/paper/main.tex --json-out paper_v2/reports/gate2_first.json
python3 "$G/figure_trace_gate.py"            --root . --manifest paper_v2/figure_trace.json      --paper paper_v2/paper/main.tex --json-out paper_v2/reports/gate3_first.json
# 各加 --final --prior paper_v2/reports/gateN_first.json → 六次全 PASS
```
期望：G1 mode1-7 全 CLEAR（mode2 "22 条引用均已验号且带回执"、mode3 "32 条 claim 0 欠账"、mode6 19 数字在 config）；G2 `ALIGNED=17`；G3 tab-1..tab-6 全 PASS（tab-6 报 CAPTION-PAIRED 对上）。

**强主张误判的已知形态**：G2 的反向覆盖检查用 `STRONG_RE` 扫正文，`\b(?:no|every|all)\s+\w+\s+(?:can|does|did|is|are|was|were|has|have)\b` 会把 B4 里"类比在此失效"的限制性陈述判成未声明强主张（报 `[undeclared:every replica has] NOT_SUPPORTED_BY_PROVENANCE`）。修法是改写措辞使其不匹配置信度词（本例 `invalidating every replica has no clean implementation` → `a cache-style invalidate-all-replicas step has no clean implementation`），不是给隐喻句补一条假声明。

**5. 视觉闸 + arXiv 封包**
```bash
python3 paper_v2/scripts/verify_visual.py     # status=PASS_WITH_NOTES, issues_remaining=[], overfull=0
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper_v2/paper l3-v7
```
期望：arXiv 封包 l3-v7 PASS — 13 页 / 0 error / 0 undefined / 0 overfull / abstract 1777 chars；tar sha256 `2bc424f0d0f6230f…`（16607 B，含 main.tex + main.bbl + 6 个 tables）。

**6. submission / 证据包重建与校验**
```bash
cd l3_pilot
python3 paper_v2/scripts/make_submission.py            # TARBALL=l3-v7.tar.gz；MEMBERS 含 tables/tab_sa_strat.tex
python3 paper_v2/scripts/make_submission.py --selftest # SELFTEST PASS (green 1 / red 2)
python3 paper_v2/scripts/make_release_pack.py          # 3012 members（n_files 已改为 len(manifest["files"])
python3 paper_v2/scripts/make_release_pack.py --verify paper_v2/submission/l3-evidence-20261122.tar.gz
```
期望：verify `{"files": 3012, "ok": true}`；bundle 的字节数与 sha256 以 `evidence_manifest.json` 的 `tarball.bytes` / `tarball.sha256` 为准，并与 `README_RELEASE.md` 首行逐字一致。**本文件自身在证据包内**，若在此写死 bundle 的字节数或 sha，会造成「改本文件→包内容变→sha 又变→文档又过期」的自指循环，因此这里只记稳定的 verify 读数（`ok=true` / `files=3012`）与下面的口径一致性判据，不记会漂移的 bundle 摘要。
口径一致性（必须四方相等）：`evidence_manifest.json` 的 `n_files`、其 `files` 条目数、tarball members、`SHA256SUMS` 行数都是 `3012`。旧版生成器在此处 `n_files=3011`（CITATION.cff 在循环后才补入清单，计数没跟着更新），校验者 `sha256sum -c` 会看到 3012 行 —— 已修并加断言，防回退。

**7. 改动可复算的三键读数（改措辞不得移动数字）**
```bash
cd l3_pilot
python3 -c "
import json
d=json.load(open('runs/l3-sa-sc-200-summary.json'))
c=d['cells']
print('cells keep STALE/n_eff:', c['keep']['STALE_FAILURE'], c['keep']['n_effective'])
print('cells del  STALE/n_eff:', c['del']['STALE_FAILURE'], c['del']['n_effective'])
print('paired n_effective:', d['paired']['n_effective'], 'diff_pp:', d['paired']['diff_pp'], 'ratio:', d['paired']['cost_ratio_del_over_keep'])
print('stratified n_pairs:', d['stratified']['n_pairs'], 'keep/del stale:', d['stratified']['keep_stale'], d['stratified']['del_stale'])
"
# 期望 keep 10/147、del 79/146；paired n_effective=146 diff=47.2603 ratio=1.2622；
#      stratified n_pairs=97 keep=5 del=30 —— B1/B2/B4 一个数都没动
```

**边界**：B1 只改标签不改数；B2 把分层从正文估计降为附录描述（正文主估计仍是全样本 +47.3pp / ×1.2622），并显式声明"分层对后处理变量条件化，不作估计量或下界"；B4 把隐喻升为命题并写清缓存类比失效边界（副本是自然语言而非寻址字节、等价按语义判定、无副本计数器 ⇒ invalidate-all 无干净实现），零新实验。页数 12→13 源于附录表。D31 已同步 DEVIATION_L3.md（前缀 977e349780a8445f→09ad3af525ba7eca，^##D=35→36）。
