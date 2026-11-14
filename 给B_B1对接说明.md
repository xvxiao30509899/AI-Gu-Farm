# 给 B：B1 本地服务怎么用

> 2026-11-14 10:10 · C 出
> 依据：`方案/B_比对报告.md:183`「B1 本地服务 | 0（C 做，B 验收）」
> 　　　`方案/B_比对报告.md:51`「B2/B3 用本地服务替代假 chat()，验证真实 HTTP 路径、JSON 解析、重试逻辑」

---

## 0 先纠正一个说法：不是一个 URL，是一个脚本

你要的「服务地址」不是我给你一个 `http://...` 让你 curl 上去——服务跑在我机器上，你的 `127.0.0.1` 指你自己，指不到我这。走内网穿透要外网、要注册、现场不稳，为一个联调用途不值得。

给你的是 `app/mockllm.py`，**你自己本机启动**，然后它就是你本机的 `127.0.0.1:8001`。

```bash
git pull          # 或从 c-final-20261114 分支拿
python -m app.mockllm
```

看到这行就成了：

```
→ MockLLM 已启动 http://127.0.0.1:8001/v1　默认场景 ok　可切 9 种
```

零依赖，只用标准库，不需要 key、不联网、不花钱。

---

## 1 你要改的只有两行配置

```json
"base_url": "http://127.0.0.1:8001/v1",
"model":    "mock-ok"
```

`model` 写真的也行（`deepseek-flash`），默认走 ok 场景。**切场景就是把 `model` 改成 `mock-<场景>`**，不用动 URL、不用重启。

---

## 2 九个场景分别测你哪一段

| `model` 写什么 | 服务返回 | 你要验证的分支 |
|---|---|---|
| `mock-ok` | 200 + 合法响应 | 主路径：HTTP 通、JSON 解析对、`content` 取到 |
| `mock-malformed` | 200 + **截断的 JSON** | 你的 `json.loads` 失败后：抛不抛？抛完记不记这一笔？ |
| `mock-notjson` | 200 + 纯文本 `OK` | 同上，更彻底（连 `{` 都没有） |
| `mock-flaky` | 前 2 次 500，第 3 次成功 | **重试是否真的生效**（详见 §3） |
| `mock-ratelimit` | 前 2 次 429，第 3 次成功 | 429 是否被当成可重试 |
| `mock-servererror` | 永久 500 | 重试耗尽后抛得清不清楚、会不会死循环 |
| `mock-badkey` | 401 | **4xx 不该重试**（key 错了重试一万次也没用） |
| `mock-truncated` | 200 + `finish_reason=length` + `content=""` | **最要紧的一条，见 §4** |
| `mock-slow` | 睡 3 秒再返回 | timeout 是否按 `config.timeout_s` 生效 |

也可以不改配置，用请求头临时切：

```bash
curl -i -X POST http://127.0.0.1:8001/v1/chat/completions \
  -H 'Content-Type: application/json' -H 'X-Mock-Scenario: badkey' \
  -d '{"model":"deepseek-flash","messages":[{"role":"user","content":"hi"}],"max_tokens":100}'
```

**请求头优先级高于 `model` 名**，所以同一个 model 下可以混测。

---

## 3 `mock-flaky` 是 B1 存在的理由

它前 2 次返回 500，**第 3 次成功并把计数清零**——正好卡在你 `config.network_retries: 2` 的边界上。

```
第 1 次请求 → 500
第 2 次请求 → 500      ← 到这里你的重试必须已经触发
第 3 次请求 → 200      ← 你的 chat() 应该正常返回，而不是抛异常
```

如果你写的是 `try: once() except: raise`，那第 1 次就抛了，你会以为是网络问题而不是自己没写重试。**这正是桩永远返回 200 时发现不了的事。**

`mock-badkey` 是它的反面：401 必须在第一次就抛。如果你在 401 上也退避重试，联调时看不出来（反正最后都失败），但正式跑时每次 key 抖一下就要白等 1.8 秒 × 调用数。

---

## 4 `mock-truncated`：C 侧昨晚真实踩过的坑

`deepseek-flash` 是推理模型：先烧 `reasoning_tokens`，再吐 `content`。`max_tokens` 给 1200 时，5 阶行列式一道题要 3700+ 推理 token，于是：

```json
{"choices":[{"message":{"content":""},"finish_reason":"length"}],
 "usage":{"completion_tokens":4000,
          "completion_tokens_details":{"reasoning_tokens":3700}}}
```

**`content` 是空字符串，HTTP 是 200。**

我们昨晚就崩在这：0/10 跑通，一开始以为是模型不会做题，查了两小时才发现是预算被推理吃光。空串往下传，会一路被记成「模型答错」，最后汇总成「能力不足」——**infra 故障被记成能力缺陷，结论就假了**。

所以你的 `chat()` 必须：

- 看 `finish_reason == "length"` 且 `content.strip()` 为空 → **抛异常，不要返回空串**
- 异常消息里带上 `reasoning_tokens`，让人一眼知道是预算问题不是能力问题
- 这个异常要能标成 `infra`，不参与能力分、不喂给学习器

参考实现：`app/llm_standalone.py:219-230`。你要自己写 `llm.py` 也行，但这段逻辑不能省。

---

## 5 参数校验也模拟了真接口

桩如果比真接口宽松，你 payload 拼错了也发现不了，到真跑才报错。所以这几种情况它返回 **400**：

- `messages` 为空或不是数组
- `messages[i]` 缺 `role`
- `max_tokens <= 0`
- 缺 `model`
- 请求体不是合法 JSON

真 OpenAI 兼容接口都是这个行为，你按 400 处理就对了。

---

## 6 排错用 `--record`

```bash
python -m app.mockllm --record /d/tmp/mockreq.jsonl
```

每个请求原样落到 jsonl。你怀疑「我明明发了 system prompt 为什么模型没照做」的时候，打开这个文件比加 print 快。

终端也会实时打一行摘要：

```
[  1] ok          model=deepseek-flash  msgs=2 max_tokens=1200
[  2] badkey      model=mock-badkey     msgs=1 max_tokens=100
```

---

## 7 三条边界，别越界

1. **它不能出正式数据。** 语义正确性一概不管，`--content` 默认就是 `"42"`。E/F 六趟必须用真模型。
2. **它只证明协议通，不证明逻辑对。** mock 永远配合你，真模型不会。
3. **它绑 `127.0.0.1`。** 别人连不上，这是故意的。

---

## 8 你要交回来什么（B 验收）

按 `B_比对报告.md:183` 那句「验收要加『同一模型』确认」，你自己验收时对着这张表打勾：

- [ ] `mock-ok` 下 `chat()` 能返回 `content`，`usage` 的 token 数能取到
- [ ] `mock-flaky` 下**不抛异常**，第三次成功返回（证明重试在跑）
- [ ] `mock-badkey` 下第一次就抛，且没有退避等待（证明 4xx 不重试）
- [ ] `mock-servererror` 下重试 2 次后抛，消息里能看出是服务端 500
- [ ] `mock-malformed` / `mock-notjson` 下抛的是「响应不是合法 JSON」而不是 `KeyError`
- [ ] `mock-truncated` 下抛的是「输出被 max_tokens 截断」且带 reasoning_tokens
- [ ] `--dry-run` 输出「理论调用数 / 含 refetch 预留数」两个数
- [ ] `config.backend` 填 `cloud` 以外的值直接报错退出，不静默降级

前六条是 B1，后两条是 B1 原定义里的第 4 点。

---

## 附：自测命令

```bash
# 一条命令看九个场景的返回码
for s in ok malformed notjson flaky ratelimit servererror badkey truncated slow; do
  printf "%-12s " "$s"
  curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    http://127.0.0.1:8001/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d "{\"model\":\"mock-$s\",\"messages\":[{\"role\":\"user\",\"content\":\"hi\"}],\"max_tokens\":100}"
done
```

预期：`ok` / `malformed` / `notjson` / `truncated` 是 200，`flaky` / `ratelimit` 第一次 500/429，`servererror` 500，`badkey` 401，`slow` 200 但慢 3 秒。

用真客户端打通全链路的测试在 `tests/test_mockllm.py`（26 条），跑一遍：

```bash
python -m pytest tests/test_mockllm.py -q
```
