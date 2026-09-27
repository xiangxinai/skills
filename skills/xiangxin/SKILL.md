---
name: xiangxin
description: 使用象信 AI 的两个模型做结构化判断——象信·系统一 xiangxin-s1（有世界知识）与象信·条件反射 xiangxin-reflex（毫秒级、便宜 100 倍、可用自己的标注数据练成，用来替代正则）——分类、打分、是非判断、路由、护栏、过滤、抽取候选选择。Use when code needs a fast, calibrated, typed decision about text or JSON (Choice / Score / Noul) via POST https://api.xiangxinai.cn/v1/systemone, when replacing regex / keyword rules with a model trained on the user's labeled data (POST /v1/reflexes), when using the `xiangxin-sdk` Python package or the `@xiangxinai/sdk` JavaScript/TypeScript package, or when the user mentions 象信 / Xiangxin / XIANGXIN_API_KEY.
---

# 象信 AI（Xiangxin）智能体 Skill

> English summary: Xiangxin has two model families behind one endpoint. System One `xiangxin-s1` ("象信·系统一", id `xiangxin-s1-1.0.0`, aliases `xiangxin-s1-latest`/`xiangxin-s1`) has world knowledge and works zero-shot. Reflex `xiangxin-reflex` ("象信·条件反射", id `xiangxin-reflex-1.0.0`; the name is regex with the g swapped for fl: "regex flash") is a small encoder with NO world knowledge, ~100x cheaper and much faster, meant to replace regex; users train their own reflex from labeled examples (`POST /v1/reflexes`) and call it as `xiangxin-reflex:<name>`. Both take a `state` (string / JSON object / array) plus a map of typed questions (`choice`, `score`, `noul`) and return calibrated probabilities in one forward pass. They never generate text. Your job as a coding agent: keep control flow in code, ask many small questions per call, and branch on probabilities/confidence. The rest of this file is in Chinese; code identifiers and field names are in English and must be used exactly.

完整文档：https://docs.xiangxinai.cn （每页加 `.md` 可取原文；索引 https://docs.xiangxinai.cn/llms.txt ，全文 https://docs.xiangxinai.cn/llms-full.txt ）。

## 1. 什么时候用象信，什么时候不用

**适合**：代码需要一个"几秒钟内一个懂行的人能拍板"的判断，且答案空间可以事先枚举：
- 工单/消息分类与路由（Choice）
- 情绪、紧急度、质量、风险按档位打分（Score）
- "这段话是否包含 X""是否满足条件 Y"之类的是非判断（Noul）
- 给大模型的输入/输出做护栏；RAG 段落过滤；从候选里挑出正确的抽取结果；把自然语言映射到函数名和枚举参数

**不适合**（请用代码或生成式大模型）：
- 生成文字、写回复、写代码、总结
- 精确计算：计数、算术、日期先后与间隔、金额比较——**在代码里做**
- 需要多步推理或大量世界知识的难题
- 象信·系统一不是编程智能体背后的聊天模型，不能替换 Claude / GPT 等作为 agent 的 LLM

### 选模型：系统一还是条件反射

两个模型用同一个接口、同样的请求和响应，只换 `model`：

| | 系统一 `xiangxin-s1`（= `xiangxin-s1-latest`） | 条件反射 `xiangxin-reflex` / `xiangxin-reflex:<名字>` |
|---|---|---|
| 知识 | 有世界知识，零样本可用 | **没有世界知识**；只会数据里出现过的规律 |
| 速度 / 价格 | 百毫秒级；¥0.042 / 百万输入 token | 毫秒级、耗时固定；¥0.00042 / 百万输入 token（便宜 100 倍） |
| 怎么得到 | 写问题即可 | 用标注样本练（练反射目前免费） |
| 长度 | 单请求 64k token | 每个问题 + state ≤ 2,048 token（超出 422，不截断） |

选择规则：
- **格式问题**（手机号、订单号、日期解析）→ 仍用正则 / 解析器，不用模型。
- **量大、重复、标准能用例子定义，且用户有（或能导出）标注数据**（工单分流、意图、垃圾/广告/敏感检测、"是否提到 X"、合规检查），或用户想替换一串正则/关键词表 → **练一个条件反射**。
- **需要常识、理解反话/言外之意、推理，或者没有数据、候选项每次都变** → **系统一**。
- **组合**：反射先跑全部流量，`confidence` 低于阈值的升级到 `xiangxin-s1`（同样的问题），再拿不准的转人工。阈值用留出的标注数据定。
- 基础 `xiangxin-reflex`（未练）可直接调用，但一般不如练过的；不确定时对比 `metrics.before` 与 `metrics.after`。

## 2. 接口速查

```http
POST https://api.xiangxinai.cn/v1/systemone
Authorization: Bearer $XIANGXIN_API_KEY        # 形如 sk-xx-...
Content-Type: application/json
```

```json
{
  "model": "xiangxin-s1-latest",
  "state": {"工单": "买的空气炸锅用了两天就不加热了，申请退货，今天能处理吗？", "订单状态": "已签收"},
  "questions": {
    "category": {"type": "choice", "instructions": "`工单` 属于哪类诉求？",
                 "criteria": {"退换货": "退货、换货、退款", "物流": "发货、配送、签收问题", "咨询": "使用方法、商品参数", "其他": null}},
    "anger":    {"type": "score", "instructions": "`工单` 中客户的不满程度", "criteria": ["平静", "有些不满", "明显生气", "非常愤怒"]},
    "urgent":   {"type": "noul", "instructions": "客户是否要求当天或尽快处理？"}
  }
}
```

响应（数值为示意）：

```json
{
  "model": "xiangxin-s1-1.0.0",
  "answers": {
    "category": {"type": "choice", "choice": "退换货", "probabilities": {"退换货": 0.91, "物流": 0.02, "咨询": 0.04, "其他": 0.03}, "confidence": 0.88},
    "anger":    {"type": "score", "score": 1.18, "legend": {"0": "平静", "1": "有些不满", "2": "明显生气", "3": "非常愤怒"}, "probabilities": {"0": 0.06, "1": 0.72, "2": 0.20, "3": 0.02}, "confidence": 0.72},
    "urgent":   {"type": "noul", "noul": 0.93}
  },
  "usage": {"input_tokens": 231, "output_tokens": 45}
}
```

- `choice`：概率最高的选项键；`probabilities` 覆盖所有选项，两位小数、和为 1；`confidence = (n·peak − 1)/(n − 1)`。
- `score`：`Σ i·pᵢ`，可以落在两档之间；`confidence = max pᵢ`。
- `noul`：答案为"是"的概率（0–1），没有 `confidence`。
- `GET /v1/models` 列出可用模型名。
- 响应头：`x-request-id`（排查问题时记录它）、`x-xiangxin-model-ms`、`x-xiangxin-total-ms`。

### Python SDK

```bash
pip install xiangxin-sdk      # 或 uv add xiangxin-sdk；Python ≥ 3.10
export XIANGXIN_API_KEY=sk-xx-...
```

```python
from xiangxin import XiangxinClient, Choice, Score, Noul

with XiangxinClient() as client:          # 读取 XIANGXIN_API_KEY / XIANGXIN_BASE_URL
    resp = client.system_one(
        state={"工单": ticket_text},
        questions={
            "category": Choice(instructions="`工单` 属于哪类诉求？",
                               criteria={"退换货": "退货、换货、退款", "物流": "发货、配送、签收", "其他": None}),
            "anger": Score(instructions="`工单` 中客户的不满程度",
                           criteria=["平静", "有些不满", "明显生气", "非常愤怒"]),
            "urgent": Noul(instructions="客户是否要求当天或尽快处理？"),
        },
    )

cat = resp.answers["category"]      # .choice / .probabilities / .confidence
anger = resp.answers["anger"]       # .score / .legend / .probabilities / .confidence
urgent = resp.answers["urgent"].noul
print(resp.model, resp.usage.input_tokens)
```

异步：`from xiangxin import AsyncXiangxinClient`，`async with AsyncXiangxinClient() as client: resp = await client.system_one(...)`。批量处理时用 `asyncio.Semaphore` 控制并发。

异常（均可从 `xiangxin` 导入）：`AuthenticationError`(401)、`InsufficientBalanceError`(402，余额不足，请去控制台充值)、`NotFoundError`(404 模型不存在)、`UnprocessableEntityError`(422 请求不合法)、`RateLimitError`(429)、`OverloadedError`(529)、`InternalServerError`(5xx)、`APIConnectionError`、`APITimeoutError`；基类 `APIError`（`.status_code`、`.detail`）< `XiangxinError`。SDK 默认对 429/529/5xx 做指数退避重试并尊重 `retry-after`（`RetryPolicy`）。**不要**再在外面套一层重试循环。

### JavaScript / TypeScript SDK

```bash
npm install @xiangxinai/sdk   # 或 pnpm add / yarn add / bun add；Node.js ≥ 18、Deno、Bun，零依赖
export XIANGXIN_API_KEY=sk-xx-...
```

```ts
import { XiangxinClient, choice, noul, score } from '@xiangxinai/sdk'

const client = new XiangxinClient()        // 读取 XIANGXIN_API_KEY / XIANGXIN_BASE_URL
const { answers, usage, model } = await client.systemOne({
  state: { 工单: ticketText },
  questions: {
    category: choice('`工单` 属于哪类诉求？', { 退换货: '退货、换货、退款', 物流: '发货、配送、签收', 其他: null }),
    anger: score('`工单` 中客户的不满程度', ['平静', '有些不满', '明显生气', '非常愤怒']),
    urgent: noul('客户是否要求当天或尽快处理？'),
  },
})
answers.category.choice      // 类型为 '退换货' | '物流' | '其他'
answers.anger.legend['3']    // '非常愤怒'
answers.urgent.noul
```

- 方法名是 `systemOne({ state, questions, model? }, { timeout?, retry?, headers?, signal? })`，`client.models.list()` 返回 `ModelCard[]`；`.withResponse()` 得到 `{ data, response, requestId }`。
- 问题直接写在调用处（或用 `choice()` / `score()`），TypeScript 才能把选项名推断成字面量；不要先赋给宽类型的变量。
- 错误类与 Python 同名（`status` 而非 `status_code`），另有 `PermissionDeniedError`(403)、`APIUserAbortError`（`AbortSignal` 取消）；`RateLimitError.retryAfter` 为秒。超时单位是毫秒（默认 30000）。
- **只在服务端使用**（Node.js / 边缘函数）。不要把 API 密钥放进浏览器代码；前端请走自己的服务端代理。SDK 在浏览器中默认拒绝创建客户端。

### 练一个条件反射（`/v1/reflexes`）

```http
POST https://api.xiangxinai.cn/v1/reflexes
Authorization: Bearer $XIANGXIN_API_KEY
Content-Type: application/json

{"name": "ticket-router", "description": "工单分流",
 "questions": {"team": {"type": "choice", "instructions": "这张工单应该转给哪个组？",
                        "criteria": {"退换货": "退货、换货、退款", "物流": "发货、快递、未收到货", "其他": "以上都不是"}}},
 "examples": [{"state": "快递三天没动静", "answers": {"team": "物流"}}, ...]}
```

- `name`：`^[a-z0-9][a-z0-9-]{0,62}$`；推理时 `model` 写 `xiangxin-reflex:<name>`。同名再提交 = **重练**（旧版本在新版本练好前照常服务；正在训练时再提交 409 `reflex_busy`）。
- `questions`：与 `/v1/systemone` 相同格式，≤ 32 个。**推理时必须用与训练时相同的 instructions / criteria**。
- `examples`：10–50,000 条，请求体 ≤ 50MB。每条 `{"state": …, "answers": {问题id: 标注}}`，可只标部分问题。标注：noul → `true/false`；choice → 选项键；score → 档位下标（0 起的整数）。state 的形式要与线上调用一致。
- 建议每个问题 ≥ 200 条、每个选项都有足够样本。≥ 20 条时随机留出 20%（最多 1,000 条）做验证。
- 响应是反射对象：`status` = `queued` → `training` → `ready` / `failed` / `cancelled`；`usable`（是否已有可调用版本）、`progress`、`metrics.before` / `metrics.after`（`accuracy`、`log_loss`、`ece`、`per_question`；`evaluated_on` 为 `val` 或 `train`）、`error`。
- 其他：`GET /v1/reflexes`（列表）、`GET /v1/reflexes/{name}`（轮询，间隔几秒）、`POST /v1/reflexes/{name}/cancel`、`DELETE /v1/reflexes/{name}`。每组织最多 20 个（409 `too_many_reflexes`）。
- 推理时反射还没练好 → 409 `reflex_not_ready`；不存在 → 404 `model_not_found`。训练服务不可达 → 503 `trainer_unavailable`（可重试）。
- 数据：样本用于练这一个反射，训练结束后最多保存 30 天（零内容留存的组织练完即删）。

```python
from xiangxin import XiangxinClient, reflex_model

with XiangxinClient() as client:
    client.reflexes.create("ticket-router", questions=QUESTIONS, examples=examples, description="工单分流")
    r = client.reflexes.wait("ticket-router")           # 轮询到 ready / failed / cancelled
    if r.status != "ready":
        raise RuntimeError(r.error)
    print(r.metrics.before.accuracy, "→", r.metrics.after.accuracy)
    resp = client.system_one(state=text, questions=QUESTIONS, model=reflex_model("ticket-router"))
```

JS：`client.reflexes.create({ name, questions, examples, description })`、`client.reflexes.wait(name)`、`reflexModel(name)`。SDK 异常：409 → `ConflictError`，413 → `RequestTooLargeError`。

给智能体的做法：先找出现成标签（工单最终处理组、审核结论、人工复核结果），整理成 JSONL；自己再留一份测试集，练完后在测试集上对比**原来的正则**与反射的准确率，并按置信度设计升级到 `xiangxin-s1` 的路径。不要编造标签；没有标签时，可以先让 `xiangxin-s1` 打标、让用户抽查后再练。

## 3. 设计问题的规则（最重要）

1. **一个问题只问一件事。** 像"分析这条消息并决定怎么处理"这种问题要拆掉：拆成几个 Choice / Score / Noul，在代码里组合。
2. **一次请求问全。** 同一个 state 的所有问题放进同一个请求——问题之间彼此隔离、并行评估，多问几个几乎不增加延迟，只多一点 token。可能用得上的问题也一起问（推测式扇出），用不上就在代码里忽略。**不要**写一个问题一次请求的循环。
3. **问题 ID 模型看不到**，完整的问题必须写在 `instructions` 里。
4. **Choice 选项的键名模型看得到**：用有意义的键（`退换货` 优于 `opt1`），描述写清楚边界；选项之间不要重叠；答案可能不在列表里时加 `其他` / `都不是`。
5. **Score 的每一档都要写成可观察的描述**（"明显生气：出现指责或威胁投诉"），2–10 档，从低到高排列。
6. **Noul 要有清晰的判定条件**。想量"程度"用 Score，不要用 Noul 的 0.5 表示"中等"。避免双重否定，`criteria.true/false` 不要和 instructions 相矛盾。
7. **结构化 state**：用 JSON 对象给每部分起名字，在 instructions 里用反引号路径指向它：`` `订单.明细[0].金额` ``。只放判断需要的内容——无关的长文本会降低准确率。
8. **instructions / criteria 也可以是对象**：把问题放一个字段，把它引用的数据放其他字段。
9. **有依赖才分两次请求**：只有当第二个请求的 state、问题或选项必须依赖第一个答案时（比如先粗分类再展开子类别选项）才串行。

## 4. 用概率和置信度控制行为

```python
# 所有问题文本和阈值集中放在一个模块里，方便人审阅和调整
AUTO_CONFIDENCE = 0.80     # 高于此值自动执行
REVIEW_CONFIDENCE = 0.50   # 低于此值直接转人工
URGENT_THRESHOLD = 0.70

ans = resp.answers["category"]
if ans.confidence >= AUTO_CONFIDENCE:
    route(ans.choice)
elif ans.confidence >= REVIEW_CONFIDENCE:
    route_with_confirmation(ans.choice)
else:
    send_to_human()
```

- 阈值随风险变化：只读操作可以宽松，涉及资金、删除、对外发送的操作要严格。
- 只想要"最可能的选项"就直接用 `choice`，不必到处设阈值；需要排序时用 `probabilities`。
- 阈值要用**用户自己的标注数据**调：准备 50–200 条带标签样例，统计不同阈值下的准确率与自动化覆盖率。
- 调好阈值后，把 `model` 固定为 `xiangxin-s1-1.0.0`，别名 `xiangxin-s1-latest` 会随新版本移动。
- 数值是"近似确定"：同一请求重复调用，中间段概率可能相差 0.01–0.02。测试里不要断言精确相等，阈值不要卡在观察到的值上。

## 5. 常用模式

- **推测式扇出**：一次请求问 10–50 个问题，代码挑需要的用。
- **置信度门控路由**：答案决定"做什么"，置信度决定"要不要自动做"。
- **组合评分**：多个原子 Score/Noul → 代码里归一化加权；改权重不改提示词。
- **意图路由**：Choice 决定交给确定性代码、专用大模型还是人工；加 `其他` 选项并用置信度兜底。
- **抽取**：模型不生成值。先用正则/解析器/大模型产生候选，再用 Choice 让象信挑；日期拆成年、月、日等枚举 Choice，拼装和比较在代码里做。
- **逐行检索**：把文档按行编号作为 Choice 选项（≤255 个，超了就分块），按概率取 top-k，再加一个 Noul 判断"文中是否有答案"。

## 6. 限制

| 项 | 值 |
|---|---|
| 单请求 | ≤ 64k token；state + 最长的一个问题 ≤ 32k token |
| Choice 选项 | ≤ 255 |
| Score 档位 | 2–10 |
| 条件反射单行 | 每个问题 + state ≤ 2,048 token |
| 速率 | 系统一：每组织默认 1,000 token/秒、300 请求/分钟；条件反射单独计：200,000 token/秒、6,000 请求/分钟（超出返回 429；服务繁忙返回 529，均应按 retry-after 退避重试） |
| 输入 | 仅文本（字符串、JSON 对象、数组）；图片/音频需先转成文字 |
| 价格 | 系统一输入 ¥0.042 / 百万 token；条件反射 ¥0.00042 / 百万 token；输出免费；练反射目前免费；注册送 ¥5（1 个月有效） |

已知短板（详见 https://docs.xiangxinai.cn/known-issues ）：长 state 效果下降；选项难以区分时偏向靠前的选项；选项键名会影响结果；数值推理弱；对提示注入等对抗内容没有专门加固；不保证"问题与其否定之和为 1"之类的结构恒等。

## 7. 隐私提示

象信 AI 不直接使用 API 与 Playground 的请求内容、答案、练反射的样本或账户数据训练模型，可能参考其类型与结构生成不含原始内容的合成数据来改进模型；请求内容与训练样本最多保存 30 天，之后自动删除，用量元数据按法律要求保存用于计费。组织可在控制台 **设置 → 组织** 开启零内容留存（不保存、不用于合成数据）。服务与数据均部署在中国大陆。详见 https://docs.xiangxinai.cn/privacy 。写代码时不要把不必要的个人信息放进 state，必要时先脱敏。

## 8. 给智能体的工作清单

1. 先问清楚：要判断什么？答案空间是什么？错判的代价是什么？
2. 把问题写成常量（一个文件集中管理问题与阈值），用户会和你一起改问题。
3. 用少量真实样例跑通（每月有 ¥5 免费额度，调用很便宜），打印 `probabilities` 看分布是否合理。
4. 看错例：如果你发现自己在解释"其实我的意思是……"，把这句话补进 instructions 或 criteria。
5. 数值、日期、计数、业务规则放在代码里；象信只负责语义判断。
6. 处理 402（余额不足）时给出清晰提示："请到 https://console.xiangxinai.cn 充值"。
7. 不要编造请求或响应字段；以本文件和 https://docs.xiangxinai.cn/api 为准。
