# 给 A：四类题集 125 道（已用你的 executor 回填完毕）

> 2026-11-13 21:50 · C 出
> 依据：技术规格 §2.1 参数边界、§2.2 后端 Task、§5 数据清单

## 状态变了：不用你回填了

上一版这份 README 让你回填 `signature` / `statement` / `reference`。**现在不需要了。**

我从 `Coder-YourAdm/AI-Gu-Farm` 取到 `app/executor.py`（指纹 `33930ae5`，含你 13:38 那次
`_ref_fraction` 改 `ValueError` 的最新版），**原样拷入、一行未改**，跑了一遍回填：

| 校验 | 结果 |
|---|---|
| `validate_task` 逐条过 | **125/125 通过** |
| `signature()` 复算去重 | 全局唯一 **125 条** |
| `judge(reference)` | 125 条全 `right` |
| `judge(reference + 1)` | 125 条全 `wrong` |
| `judge(拒答 / 格式错)` | 全 `unanswered` |
| 十进制容差路径 | 36 题全 `right` |
| 重跑幂等 | 四个数据文件逐字节相同 |

判分内核只能有一个版本，所以我没有另写一套 `signature()` —— 这是这份数据能用的前提。

## 四类题集

| 文件 | 数量 | 组成 |
|---|---|---|
| `calibration.json` | 20 | integral 10（poly/gamma/gaussian_odd = 7/7/6）+ det 10 |
| `common_dev.json` | 80 | 20 组（`gen` 0–19），每组 integral 2 + det 2 |
| `challenge_pool.json` | 5 | integral 3 + det 2 |
| `holdout.json` | 20 | integral 10 + det 10 |

三类积分全卷 21/21/21（§5 要求「尽量均衡」），det 62 道，全部非奇异。
生成种子固定 `20261113`，`manifest.json` 记了每份文件的 sha256 与 executor 指纹。

## 你要做的只有一件事：抽验

不需要跑全量。三条命令，30 秒：

```bash
# 1. 边界：125 道逐条过你的 validate_task
pytest tests/test_contract_compat.py -q

# 2. 数据契约：签名互斥、manifest 与文件一致
pytest tests/test_datasets.py -q

# 3. 手验一条
python -c "import json;d=json.load(open('data/calibration.json'));t=d[0];print(t['statement']);print(t['reference'])"
```

第 1 条里 `test_known_bypass_entries_still_bypass` 是**故意留的例外检查**：V07 越界这件事
被登记在 `KNOWN_BYPASS` 里，哪天你把用例改了，这条会红并提醒删除登记 —— 不靠人记。见下。

## 十道验证题 + 一个需要你定的问题

`data/verify_bank.json` + `十道验证题_给A核对.md`：附录 A 的 I1/I3/I4/I5/I7、D1/D2/D3/D6/D7。
喂正解必须 `right`、喂错解 `wrong`，**失败条数要求 0**。

**其中 V07 是规格自相矛盾的地方**，详见 `给A_边界冲突说明.md`：

> 附录 A 的 D2 用例 `[[1,2,3],[4,5,6],[7,8,10]]` 里元素 `10` 超出 §2.1 的 `[-9,9]`。
> 你的 `test_D2` 走 `_det() → solve_exact()`，绕过了 `validate_task()`，所以 36 条全绿也抓不到。
> **这不是你写错**，是规格前后不一致。要你定的是「留作算术单测还是换用例」。

需要你选一个（都不阻塞）：

1. 保留 D2 当 `bareiss` 单测，注释写明「越界输入，仅测算术，不用于出题流程」← 我倾向这个
2. 换合法矩阵。注意不能简单把 `10` 改 `9`：`det([[1,2,3],[4,5,6],[7,8,9]]) = 0`，会撞「非奇异」要求
3. 附录 A 加一行注：D2 为算术用例，不代表合法参数域

## 我这边还等你一样东西

`web/result.json`（留出评测结果）。字段契约在 `app/result_schema.py`，四条硬约束：

| 约束 | 原因 |
|---|---|
| `is_mock` 必填 | 假数据必须自我声明，页面据此挂红标，防止 mock 冒充真结果 |
| `holdout.total == metric.n_tasks` | 两处各写 20，改一处就静默不一致 |
| 同 key 祖先跨趟同标签同色 | 换名字跨趟对照就废了 |
| `evidence[].run_id` 必须真实存在 | 每个数字要能追溯回一趟运行 |

红线：`reference` / `statement` / `signature` / `template` / `params` / `raw_response`
一律不许进 `result.json`（`tests/test_result.py` 有测试扫）。

不给也能演示 —— 结果页现在吃 mock 数据并挂红标。但正式提交前这份得是真的。

## 一条红线，重复一次

`signature()` 只能有一套实现。数据里哪道题的参数形式你不接受，
**告诉我改数据，不要改校验函数去迁就数据**——数据错可以重生成，签名规则改了全盘皆错。
