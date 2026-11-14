# 题域饱和实测结论（给 A / B）

时间：2026-11-14 00:20–01:10 · 执行：C
模型：`deepseek-flash`（DeepSeek 官方）+ `Qwen/Qwen3.5-9B`、`Qwen/Qwen3.5-4B`（硅基流动）

## 一句话

**这套题域对 flash 已经没有难度梯度。** 我能拿到的所有"失败"都是预算被推理吃光
（`finish_reason=length`），不是模型不会。把它算成能力失败，会导出一条假的退化曲线。

## 实测数据（全部可复跑，命令在末尾）

| 题目类型 | 配置 | 结果 | 判读 |
|---|---|---|---|
| calibration 20 题 | flash, 6000 tok | **20/20** | 天花板 |
| calibration 20 题 | flash, 4000 tok | 19/20 | 唯一失败是截断 |
| calibration 20 题 | flash, 1200 tok | 14/20（6 题 `unanswered`） | **预算假象**，非能力 |
| gamma_sum 变体 L1–L4（附录 B） | flash, 6000 tok | **16/16** | 变体没增加难度 |
| 5 阶行列式 | flash, 6000 tok | 10/10（单题 ~20s / 4.5k tok） | 慢但全对 |
| 6–7 阶行列式 | flash, 6000 tok | **0/6，全部 `finish=length`** | 预算不够，不是不会 |
| 5 阶大数值（\|元素\|≤99） | flash, 6000 tok | **0/6，全部 `finish=length`** | 同上 |
| 提示词"省流：只给答案" | flash, 6000 tok | **8/8** | 提示词策略压不出差距 |
| 提示词"标准：逐步求解" | flash, 6000 tok | **8/8** | 对照组同样全对 |

硅基流动两个小模型（同一批题）：

| 模型 | 通过率 | 延迟 | 结论 |
|---|---|---|---|
| Qwen3.5-9B | 8/8 = 100% | 中位 **35s**，长尾 160s | 也到顶，且太慢 |
| Qwen3.5-4B | 4/8 = 50% | 中位 **180s 超时**（4 次 timeout） | 不可用 |

20 代演化需要 ~700 次调用。按 9B 的 35s/题、并发 10 算，约 **40 分钟**；
按 4B 的超时率，跑不完。

## 三条结论

1. **附录 B 的 gamma_sum 变体没解决问题。** 它的假设是"模板太好识别"，但
   `∫₀^∞ xⁿe^{-rx}dx = n!/r^{n+1}` 是闭式公式，模型直接套，加项数只是加算术量。
   实测四项七次幂仍然 100%。
2. **能压住 flash 的只有"算不动"，不是"不会算"。** 6–7 阶行列式和大数值 5 阶
   都栽在 token 预算上。这类失败必须标 `infra`，否则学习器会从自己的预算不足里
   归纳出"行列式算法不行"的假规则（这个坑 11-13 已经踩过一次）。
3. **今晚没有便宜的"弱而快"模型。** 弱的那两个要么也全对（9B），要么超时（4B）。

## 给 A/B 的建议（按代价排序）

| 方案 | 需要谁动 | 代价 | 能否今晚做完 |
|---|---|---|---|
| A. 接受 `no_change`，页面如实写"该题域已饱和" | 无人 | 0 | 已完成 |
| B. 换一个"弱且快"的 solver（≤10s/题，通过率 30–70%） | 出资方/A | 未知 | 不能 |
| C. 放开题域上限（6 阶以上行列式、更大数值），并**同步抬高 token 预算** | A 改 `validate_task` + config 预算 | 中 | 不能 |
| D. 让 executor 支持 `gamma_sum` | A | 小 | 可以，但收益近 0（见结论 1） |

**我的判断：C 是唯一能真正做出上升曲线的路，但必须先抬预算，否则 6 阶只会
继续 `finish=length`。** 单纯放开阶数不提预算，等于把"不会"换成"没写完"。

## 复跑命令

```bash
export LLM_API_KEY=<deepseek key>

# 1. 题域难度（calibration / 变体 / 高阶行列式）
python tests/probe_model.py --model deepseek-flash --base-url https://api.deepseek.com/v1 \
  --key-env LLM_API_KEY --data data/calibration.json --n 20
python tests/probe_model.py --model deepseek-flash --base-url https://api.deepseek.com/v1 \
  --key-env LLM_API_KEY --data data/hard_bank.json --n 16
python tests/probe_model.py --model deepseek-flash --base-url https://api.deepseek.com/v1 \
  --key-env LLM_API_KEY --data data/hard_det.json --n 6

# 2. 硅基小模型
export SILICON_KEY=<siliconflow key>
python tests/probe_model.py --model "Qwen/Qwen3.5-9B" --n 8
python tests/probe_model.py --model "Qwen/Qwen3.5-4B" --n 8

# 3. 20 代真跑（约 700 次调用）
python -m app.evolve_run --run-id E_real_20261114 --export
```

题集由脚本生成，可复现：
`tests/make_gamma_bank.py`（gamma_sum 四档）、`tests/make_hard_det.py`（6–7 阶 / 大数值 5 阶）。

---

## 附：20 代真跑结果（2026-11-14 00:40–01:10）

`run_id = E_real_20261114` · `model = deepseek-flash` · 6 个体/代 · 21 代（g0–g20）

| 指标 | 数值 |
|---|---|
| 累计调用 | **724 次**（含 g15 补测 10 次） |
| 输入 / 输出 token | 106,662 / **546,805**（输出里大部分是推理 token） |
| 祖先题测量点 | 63 个（21 代 × 3 族），**62 个为零失败**，1 个 0.333（g12 的行列式族） |
| 个体总分 | 每代 6 人全部 7.00（solve 5 + set 2），无区分度 |
| 冠军 | G01 → 后段 G03（同分并列，按 pop 顺序） |
| 学到的规则 | 1 条（g19，来自某代共同题的一处失败） |

**g15 有一次数据事故，已按可审计方式修正**：该代运行时进程被系统挂起（wall_ms
记录为 27,088,316ms ≈ 7.5 小时），10 道祖先题全部超时。已用同一冠军基因**补测一次**
（结果仍为 0 失败），并把该代 `wall_ms` 置为相邻代均值；修正说明写进该代
`warnings` 字段，不静默改数。

大屏现状：`web/history.json` 的 `is_mock=false`、`run_id=E_real_20261114`，
页面显示 REAL 标；平线下方有一句由数据推导的说明（"63 个测量点里 62 个为零失败"），
避免被误读成"数据没加载"。

**结论没变**：这套题域对 flash 已接近饱和，20 代里没有可学的失败，所以曲线是平的。
这不是坏数据，是真实的能力天花板。
