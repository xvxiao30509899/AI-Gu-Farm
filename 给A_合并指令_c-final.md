# 给 A：合并指令（C 侧交付，2026-11-14）

## 一句话

C 的成果已经在远端分支 **`c-final-20261114`**，它正好长在 `main` 之上，
**你那边是一条 `git merge` 快进，不会有冲突**。

```bash
git fetch origin
git checkout main
git merge origin/c-final-20261114      # 预期输出：Fast-forward
pytest                                  # 预期：274 passed, 1 skipped
```

不需要 rebase、不需要手工解冲突、不需要按文件挑拣。

## 为什么这次不会冲突

上一轮的问题不是「忘了 pull」，而是两条历史**从来没有共同祖先**：

```
git merge-base --all HEAD origin/main   → 空
```

本地那个分支是从 `27b69d63` 拉的，而这个 commit 在远端根本不存在。
所以当时照「先 pull 再合」操作，只会得到 `refusing to merge unrelated histories`。

这次的做法是：以 `origin/main`（`a110cd6`）为基准重新 clone，
把 C 的成果重新提交上去，**本地那 9 个孤立提交直接放弃**。
现在 `git merge-base origin/main origin/c-final-20261114` = `a110cd6`，是真正的祖先。

## 合并后会多出什么

只有新增，**没有修改你的任何文件**：

| 路径 | 说明 |
|---|---|
| `app/demo.py` | 双跑编排 `pair` / `rollback` / `status` |
| `app/evolve_run.py` | 20 代演化运行器 |
| `app/evaluate.py` | 留出评测 3 趟 × 3 血脉 × 20 题 |
| `app/export.py` | 白名单导出（`reference` 不外泄） |
| `app/webdata.py` | `web/*.json` → `web/data.js`，支撑 `file://` 双击 |
| `app/serve.py` | 标准库静态服务 |
| `app/llm_standalone.py` | 真模型调用，token 预算读 `config.json` |
| `app/evolve_schema.py` / `app/result_schema.py` | 两份数据契约校验 |
| `data/` | 125 题 + 三套硬题库 + `holdout.jsonl` |
| `prompts/` | 出题 / 解题 / 六份种子基因 / 交叉 / 变异 |
| `web/` | 三张单文件页面 + `data.js` + 三份真数据 |
| `tests/` | 226 条 C 侧测试（与你的 48 条合计 274） |
| 若干 `.md` | 交付说明、实测结论、复核表 |

`app/executor.py` **没有改动**。我核对过：`diff --strip-trailing-cr` 对本地副本与
你的 `a110cd6` 版本差异为 0 行，只有行尾不同。仓库里用的是你的原文件。

## 请你复核的三件事

1. **`web/data.js` 是生成物**，别手改。它由 `app/webdata.py` 从 `web/*.json` 打包，
   改了页面会跟数据不一致。
2. **`data/` 里的 125 题**是我按技术规格 §2 造的，`reference` 由你的 `executor.solve_exact()`
   现算回填，`validate_task` 125/125 通过。但**题的难度分布**值得你看一眼——
   详见 `给AB_题域饱和实测结论.md`，这是当前最大的风险。
3. **`holdout.jsonl` 只有 `id/template/params`，没有参考值**，评测时用
   `solve_exact()` 现算（20/20 成功）。如果你希望留出题也走 `data/holdout.json`
   的完整字段，需要你确认一次口径。

## 两个需要你决策的缺口

1. **挑战题从未注入**。`evolve_run.py` 每代的 `warnings` 都写着
   `slots.challenge=0`（21 代全部）。大屏 03 区那行「本次运行没有注入挑战」
   是真实数据缺口，不是渲染失败。需要 B 的 challenge 队列。
2. **T0 至今没有定义**。`方案/分工方案_v7.md:274` 原文是
   「T0 = 开赛时刻（待确认钟点）。」——等于空值，整条时间轴（22H 锁协议）
   都挂在上面。建议改成绝对时间，例如：12:00 前定题域/模型，
   12:00–18:00 重跑 pilot，18:00–22:00 正式 pilot。

## 如果合并出意外

先别动手改，直接告诉我实际报错。最可能的两种：

- `! [rejected]` 或 `non-fast-forward`：说明你的 `main` 在我这次 push 之后
  又前进了。这种情况我能用一次 rebase 解决，不需要你动。
- `error: Your local changes would be overwritten`：你工作区有未提交改动，
  `git stash` 后再 merge。
