# 给 B：prompts 五份 + 一条会直接崩的渲染约束

> 2026-11-13 21:50 · C 出
> 依据：技术规格 §3.2 setter、§3.3 solver、§3.4 算子约束、§3.5 六份基因
> 配套：本包内 `一页纸_对接.md` 第一节就是你要交的东西，其余可最后再看

## 五份文件

| 文件 | 来源 | 说明 |
|---|---|---|
| `prompts/setter.md` | **§3.2 原文**（脚本抽取，非手抄） | 出题 system prompt，末尾 `{set_prompt}` |
| `prompts/solver.md` | **§3.3 原文**（脚本抽取） | 解题 system prompt，含两个占位符 |
| `prompts/seed_genotypes.json` | **§3.5 原文**（六份基因全文） | 初代种群，`origin="seed"` |
| `prompts/crossover.md` | §3.4 约束**起草** | 占位符名待你确认 |
| `prompts/mutate.md` | §3.4 约束**起草** | 占位符名待你确认 |

前两份和基因文件是**从规格代码块自动抽取**的，不是手抄。`tests/test_prompts.py`
会重新抽一次做逐字比对——有人改一个字的措辞，测试立刻红。因为改一个字就可能
改变模型行为，而 pilot 的数字会因此不可复现。

## ⚠️ 一条会直接崩的约束：只能用 str.replace()

`setter.md` 里有题面模板语法：

```
poly: params={coeffs:[a0,...,ad],lower:l,upper:u}
```

这串花括号会被 Python 的 `str.format()` 当成占位符。实测：

```
setter_md.format(set_prompt="优先 poly")
→ KeyError: 'coeffs'
```

**规格原文就是这样，不能改**（改了就和 §3.2 不一致）。所以五份 prompt 全部只能用
`str.replace()` 渲染：

```python
# ✅ 对
text = md.replace("{set_prompt}", genes["set_prompt"])

# ❌ 错 —— KeyError: 'coeffs'
text = md.format(set_prompt=genes["set_prompt"])

# ❌ 错 —— f-string 同样会解析花括号
text = f"前缀{setter_md}"
```

已钉成断言：`tests/test_prompts.py::test_must_render_with_replace_not_format`。
谁改回 `.format()`，测试立刻红。

## 占位符清单

| 文件 | 占位符 | 替换规则 |
|---|---|---|
| `setter.md` | `{set_prompt}` | 当前个体的 `genes.set_prompt` |
| `solver.md` | `{solve_prompt}` | 当前个体的 `genes.solve_prompt` |
| `solver.md` | `{check_instruction}` | `self_check=true` → 「在内部检查符号、数量级和算术，再给最终答案。」；`false` → 空串 |
| `crossover.md` | `{parent_a}`、`{parent_b}` | **名字待你确认** |
| `mutate.md` | `{parent}`、`{field_to_mutate}` | **名字待你确认** |

后两个是我按规格命名风格暂定的。**如果和你 `orchestrator.py` 的渲染代码不一致，
告诉我改名，Ctrl+H 一分钟的事。** 不要为了迁就我的命名去改你的算子逻辑。

## 两份算子的输出约定

都要求模型只输出：

```json
{"genes":{"solve_prompt":"...","set_prompt":"...","self_check":true}}
```

`self_check` 必须是**真布尔**，不是 `"true"` 字符串——`tests/test_prompts.py`
里有断言检查初代六份都是 `type(v) is bool`。

## §3.5 的三条约束已写成测试

```
每份 len(solve_prompt) + len(set_prompt) <= 200   实测 76–123
六份 solve_prompt / set_prompt 各自两两不同
self_check 3 true / 3 false                       保证变异第一次翻转能产生新值
```

跑一下：

```bash
pytest tests/test_prompts.py -q     # 23 条
```

## 我需要你给的两个东西

```python
# app/llm.py
def complete(messages: list[dict]) -> tuple[str, int, int]:
    """返回 (text, tokens_in, tokens_out)。并发由调用方控，我这边固定 6。"""
```

外加 `config.json` 里的 `model` 和 `base_url`（现在是 `REPLACE_WITH_*` 占位）。
有这两个我就能把双跑演示从 mock 切成真数据，三行进化说明里的数字就能换成真的。

## 本地推理不用等了

Ollama / `11434` 那条路已取消。实测 ModelScope 78.5 KB/s、GitHub 43 KB/s，
5.2 GB 要 20 小时。`config.backend` 只有一个合法值 `cloud`，`local` 分支保留字段
但不填、不测、不启用。**你不需要等任何局域网地址。**
