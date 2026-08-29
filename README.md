# KuaiRand-Pure Starter Kit

## 依赖

Python 3.9+ 和 numpy。**没有别的。** 不需要 torch、pandas、sklearn。

## 数据

从 https://kuairand.com 下载（Zenodo 直链，无需注册）：

```bash
# 在 Starter Kit 目录下执行，解压后得到 ./KuaiRand-Pure/
wget https://zenodo.org/records/10439422/files/KuaiRand-Pure.tar.gz
tar xzf KuaiRand-Pure.tar.gz
```

## 运行

```bash
python3 baseline.py --model fm
```

`--data_dir` 默认 `./KuaiRand-Pure/data`；数据放在别处时显式指定。

`--model` 可选 `fm`（官方 baseline）/ `pop`（trivial baseline）/ `random`（下界，用于自检评测代码）。
FM 全程约 40 秒（CPU，单核）。

## 任务定义（口径已写死，不要改）

| | |
|---|---|
| 任务 | **用户内排序** —— 每个用户只对其在评测集中的曝光排序，不做全库检索 |
| 相关性标签 | `long_view`（原生列，0/1） |
| 指标 | `GAUC`、`nDCG@5`；**主分 = 两者平均** |
| 数据划分 | train `20220408–20220421` / valid `20220422–20220428` / test `20220429–20220508` |
| 零正例用户 | nDCG 记 0.0 并计入平均；GAUC 只统计 `0 < 正例数 < 曝光数` 的用户，按正例数加权 |
| nDCG gain | `2^rel − 1`（二元标签下等价于 identity） |

实现见 `evaluate.py`，全部约定写在文件头注释里。

## Baseline 阶梯

test 集上的分数。**要打败的是 FM 这一行。**

| | GAUC | nDCG@5 | primary |
|---|---|---|---|
| random（下界，自检用） | 0.4996 | 0.4511 | 0.4753 |
| item popularity（trivial） | 0.6308 | 0.5121 | 0.5715 |
| **FM（官方 baseline）** | **0.6610** | **0.5282** | **0.5946** |

### ⚠️ 指标的真实区间：nDCG@5 的天花板是 0.729，不是 1.0

test 集 23,875 个用户里：

| | 占比 | 对指标的影响 |
|---|---|---|
| 全负用户（该用户所有曝光都不是 long_view） | **27.1%** | nDCG 恒为 **0**，任何模型都救不了；不计入 GAUC |
| 全正用户 | **9.2%** | nDCG 恒为 **1**；不计入 GAUC |
| 有区分度的用户 | **63.7%** | GAUC 的实际样本 |

所以用真实标签当预测分（oracle，完美排序）也只能拿到：

| | random | FM baseline | **oracle 上限** | FM 已吃掉的区间 |
|---|---|---|---|---|
| GAUC | 0.4996 | 0.6610 | **1.0000** | 32.3% |
| nDCG@5 | 0.4511 | 0.5282 | **0.7289** | 27.8% |
| **primary** | 0.4753 | **0.5946** | **0.8645** | **30.7%** |

**评估进展请以 oracle 为分母。** 看到 0.5946 就以为「离满分 1.0 还很远」是误判——
baseline 已经吃掉可用区间的三成，剩余 headroom 是 0.27 而不是 0.41。

FM 在 5 个随机种子上的 std 均为 **0.0008**。据此收敛判据取 **ε = 0.002（≈2.5σ）, N = 3**：
连续 3 轮迭代 validation 主分提升不超过 0.002 即判定收敛。

> 自检：如果你的评测代码跑 `--model random` 得不到 primary ≈ 0.475（±0.001），说明 harness 有问题，先修它。

## 提交格式

CSV，含表头，一行对应评测集的一行：

```
row_id,user_id,video_id,score
0,0,7531,-3.34176
1,0,4214,-1.4955
...
```

| 字段 | 说明 |
|---|---|
| `row_id` | 0 起连续递增，对应 `data.load()[split]` 的行序（确定性：先读 `log_standard_4_08_to_4_21_pure.csv` 再读 `log_standard_4_22_to_5_08_pure.csv`，按 date 过滤后保持原文件顺序） |
| `user_id` / `video_id` | 冗余字段，仅用于校验对齐 |
| `score` | 你的模型给该行打的分，任意实数，只用相对大小；不允许 NaN / Inf |

> **为什么必须带 `row_id`：** `(user_id, video_id)` 在评测集里**不唯一** ——
> test 集有 3.06% 的重复对，最多重复 12 次。所以它不能作为主键。

生成与校验：

```bash
python3 submit.py --make  --split test  submission.csv    # 用官方 FM baseline 生成一份示例提交
python3 submit.py --check --split test  submission.csv    # 校验格式与对齐
python3 submit.py --score --split valid submission.csv    # 校验并打分（本地 valid 可用）
```

`--check` 会拒绝：表头错误、行数不符、`row_id` 跳号、`user_id`/`video_id` 与评测集不对齐、
`score` 非数字或为 NaN/Inf。**提交前请自行跑一遍 `--check`。**

## 从哪里开始改

下面的排序是**实测过的**，不是猜的。组委会已经试过的死路直接标出来，别重复踩。

### 已实测：这两条没有收益，不要浪费迭代

| 试过的 | 结果 |
|---|---|
| **加静态特征** —— 把 CWM 的 13 个特征域全接进来（+`music_id`/`video_type`/`upload_type` + 6 个用户侧粗桶） | primary **0.5940** vs 5 域的 **0.5950**，噪声内无差别，甚至略降 |
| **加模型容量** —— embedding 维度 k = 8 / 16 / 32 | 0.5895 / 0.5902 / 0.5887，几乎不动 |

原因：`user_id × video_id` 的交叉已经吃掉了大部分可学的信号。`follow_user_num_range` 这类粗桶
在 `user_id` 面前是冗余的；而 114 万行数据也撑不起更大的容量。**瓶颈不在特征和容量。**

⚠️ 另外注意：**纯用户侧特征的一阶项对分数贡献恒为 0。** 因为排序在用户内部做，任何在用户内为常数的项
都不改变组内顺序（实测：`item_pop × 用户偏置` 和纯 `item_pop` 的分数一位不差）。用户侧特征只能通过
**与物品侧的交叉项**起作用。

### 未探索：headroom 应该在这里

按我们判断的可能性排序（**这几条组委会没测过，是留给你们的**）：

1. **换损失函数。** 现在是 pointwise logloss，但指标（GAUC / nDCG）是**排序指标**。
   换成 pairwise（BPR）或 listwise（对该用户的曝光做 softmax）—— 目标函数和评测口径对齐，
   这是我们认为最可能有效的一条。
2. **用户历史序列。** 现有特征**完全没用到行为序列**。KuaiRand 每用户在 train 里有上百到上千条交互，
   DIN / SIM 那一类的兴趣建模是完全空白的方向。
3. **多目标。** 日志里还有 `is_click`、`is_like`、`is_follow`、`is_comment`、`is_forward`、`play_time_ms`，
   可以做多任务辅助 `long_view` 主任务。
4. **观看时长的建模。** [CWM](https://github.com/hyz20/CWM) 的贡献正是这条：它把观看时长做**删失回归**
   （视频播完时真实观看时长被截断，所以用单侧损失而非平方误差）。这是个有研究深度的方向。
5. **换模型。** DeepFM / DCN / xDeepFM。鉴于容量实测不是瓶颈，**优先级放在 1-4 之后**。
6. **时间特征与分布漂移。** `hourmin`、`date`，以及 train 与 test 之间的漂移。
7. **无偏验证（进阶）。** `log_random_4_22_to_5_08_pure.csv` 是随机曝光日志（118 万行），
   可作为额外的无偏验证集，检查模型是否只在有偏流量上过拟合。

## 用你自己的模型（包括 CWM）

`evaluate.py` 与模型完全解耦，它只要三个等长数组：

```python
from evaluate import evaluate
print(evaluate(user_ids, labels, scores))   # scores 可以来自任何模型
```

- `user_ids`：评测集每一行的 user_id
- `labels`：该行的 `long_view`（0/1）
- `scores`：你的模型给该行打的分（任意实数，只用相对大小）

所以你可以完全不用 `baseline.py`，换成 PyTorch、LightGBM 或 [CWM](https://github.com/hyz20/CWM) 的 xDeepFM，
只要最后把 `scores` 交给 `evaluate()` 即可。**评分口径由 `evaluate.py` 唯一决定。**

> 用 CWM 需注意：它依赖 `torch==1.6.0`（2020 年版本，新 GPU 上大概装不上），
> 且它的损失优化的是 counterfactual watch time、评测标签是自己重建的 `long_view2`。
> 它是一篇时长纠偏论文的研究代码，可以当**进阶参考**，不建议作为起步点。

## 文件

| | |
|---|---|
| `evaluate.py` | 指标实现 + 全部口径约定。**不要改。** |
| `data.py` | 数据加载、官方划分、特征编码。加特征改这里。 |
| `baseline.py` | 三个 baseline。FM 是要打败的那个。 |
| `baseline_scores.json` | 官方发布的分数 + 种子方差 + 收敛参数。 |
| `submit.py` | 生成 / 校验提交文件。 |
| `ablation_features.py` | 特征消融实验，可复现「加特征没有收益」那组数字。 |
