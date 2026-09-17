# OpenAI API 调用入门

- 目标读者：有 Python 基础，未使用过 LLM API
- 前置知识：HTTP 请求、JSON、环境变量、字典与列表
- 学习时长：1.5 小时 / 可跳过：第 6 节（坑点）可在遇到时再回查

---

## 第 1 节：它是什么

**OpenAI API** 是一个 HTTP 接口，你发送一段文本（问题、指令、对话历史），它返回 AI 生成的回复。

**类比**：像给一个很聪明的远程助手发邮件——你写明任务和上下文，他回复结果；只不过这个"邮件"是 JSON 格式，"助手"是 GPT 模型。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 不用它会怎样？

如果要让程序"理解文本""生成内容""提取信息"，传统做法是：
- 写一堆正则表达式匹配关键词（一改就坏，覆盖不全）
- 手工标注几千条数据训练模型（成本高、周期长）
- 写死规则和模板（灵活性为零）

**痛点**：每增加一个新场景，就要重写一套规则；文本稍微变化，程序就失效。

### 问题根源

**语言理解能力被硬编码在代码逻辑里**，而自然语言本身是灵活、开放的，规则覆盖不了所有变体。

### 经典应用场景

- 非结构化文本提取（发票、简历、合同 → 结构化数据）
- 内容生成（文章改写、代码注释、邮件回复）
- 对话式产品（客服机器人、智能助手）
- 数据清洗与分类（自动打标签、情感分析）

---

## 第 3 节：读下去之前，先搞懂这些概念

| 术语              | 大白话解释                                                | 示例                                        | 对应正文节     |
| --------------- | ---------------------------------------------------- | ----------------------------------------- | --------- |
| **API Key**     | 你的身份证，证明你有权限调用 API                                   | `sk-proj-abc123...`                       | 第 4 节示例 1 |
| **messages**    | 对话历史，告诉模型"之前说了什么"                                    | `[{"role": "user", "content": "你好"}]`     | 第 4 节示例 1 |
| **role**        | 消息发送者角色：`system`（系统指令）、`user`（用户）、`assistant`（AI 回复） | `role="system"` 表示"给 AI 的总指令"             | 第 4 节示例 2 |
| **temperature** | 回复的随机性：0=固定、2=极度发散                                   | `temperature=0` 适合提取数据                    | 第 4 节示例 2 |
| **max_tokens**  | 回复最多生成多少个 token（约 0.75 个英文单词）                        | `max_tokens=500` 大约 375 个英文词              | 第 4 节示例 2 |
| **JSON mode**   | 强制模型返回合法 JSON                                        | `response_format={"type": "json_object"}` | 第 4 节示例 3 |

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本

**目标**：发送一条消息，拿到 AI 回复。

```python
import os
from openai import OpenAI

# 初始化客户端，从环境变量读取 API Key
# 需要提前在终端执行：export OPENAI_API_KEY="sk-proj-..."
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 发送请求：messages 是对话历史列表
response = client.chat.completions.create(
    model="gpt-4o-mini",  # 模型名称，决定能力和价格
    messages=[
        {"role": "user", "content": "用一句话解释什么是 API"}
    ]
)

# 提取回复内容
answer = response.choices[0].message.content
print(answer)
```

**逐段解读**：

1. **导入与初始化**：
   - `OpenAI()` 创建客户端对象，负责发送 HTTP 请求
   - `api_key=os.getenv(...)` 从环境变量读密钥（避免硬编码泄露）

2. **构造请求**：
   - `model` 指定用哪个模型（`gpt-4o-mini` 便宜快速，适合大批量调用）
   - `messages` 是列表，每个元素是一条消息（字典格式）
   - `role="user"` 表示这是用户发的话，`content` 是具体内容

3. **解析响应**：
   - `response.choices[0]` 取第一个候选回复（通常只有一个）
   - `.message.content` 就是 AI 生成的文本

**这一步你得到了什么**：能成功调用 API 并打印回复，验证 Key 和网络环境正常。

---

### 示例 2：加 system prompt 和参数控制

**目标**：让 AI 扮演特定角色，并控制回复风格。

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        # system 消息：给 AI 的"人设"和总指令，它会贯穿整个对话
        {"role": "system", "content": "你是一个专业的数据分析师，只用数据和事实说话，不说废话。"},
        {"role": "user", "content": "分析一下这段文本的情感：'这个产品太烂了'"}
    ],
    temperature=0,      # 设为 0 让输出确定性强，适合提取任务
    max_tokens=100      # 限制回复最多 100 token，避免超预算
)

print(response.choices[0].message.content)
```

**逐段解读**：

- <mark style="background: #BBFABBA6;">**system 消息的作用**：类似"给 AI 发的工作说明书"，它会影响后续所有回复的风格和行为。写在 `messages` 列表最前面。</mark>
- <mark style="background: #BBFABBA6;">**temperature=0**：回复变得固定（每次调用结果几乎一致），适合需要稳定输出的场景（如数据提取）。设为 1-2 则更有创意，但不可控。</mark>
- **max_tokens=100**：防止回复过长导致计费暴涨。1 token ≈ 0.75 个英文单词，中文约 1 token = 1 个字。

**和我直觉相反的点**：`temperature=0` 不是"降低质量"，而是"锁定输出"——当你需要每次提取同样格式的数据时，这是必须项。

**这一步你得到了什么**：能定制 AI 行为，输出变得可控、稳定。

---

### 示例 3：返回 JSON 格式

**目标**：让 AI 返回结构化数据，方便程序解析。

```python
import os
import json
from openai import OpenAI

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "你是数据提取助手，只返回 JSON 格式，不要任何额外文字。"},
        {"role": "user", "content": '从这段话提取信息："张三，手机号 13800138000，购买了 3 个苹果"。返回格式：{"name": "...", "phone": "...", "item": "...", "quantity": 数字}'}
    ],
    temperature=0,
    response_format={"type": "json_object"}  # 强制返回合法 JSON
)

# 解析 JSON
result = json.loads(response.choices[0].message.content)
print(result)  # {'name': '张三', 'phone': '13800138000', 'item': '苹果', 'quantity': 3}
print(f"购买人：{result['name']}, 数量：{result['quantity']}")
```

**逐段解读**：

- <mark style="background: #BBFABBA6;">**response_format**：告诉模型"无论如何都返回合法 JSON"</mark>，避免返回 `输出如下：{"name":...}` 这种带前缀的文本（会导致 `json.loads()` 报错）。
- **<mark style="background: #BBFABBA6;">prompt 里指定格式</mark>**：虽然有 `json_object` 约束，仍需在 prompt 里写清字段名和类型，否则模型可能自由发挥字段名。
- **<mark style="background: #BBFABBA6;">json.loads() 解析</mark>**：把字符串变成 Python 字典，之后就能像普通数据一样用 `result['name']` 访问。

**这一步你得到了什么**：可以把 LLM 当"智能正则表达式"——输入文本，输出结构化数据，直接对接下游程序（pandas、数据库、Excel）。

---

## 第 5 节：最小可用的实际用法

以下是一个**生产场景模板**：批量提取非结构化文本信息并保存为 CSV。

```python
import os
import json
import csv
from openai import OpenAI

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 待处理的文本列表（实际场景可能从文件、数据库读取）
texts = [
    "订单001：李四购买了5个橙子，联系电话13900139000",
    "订单002：王五买了2箱牛奶，电话13700137000"
]

results = []

for text in texts:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "提取订单信息，返回 JSON：{\"order_id\": \"\", \"name\": \"\", \"item\": \"\", \"quantity\": 数字, \"phone\": \"\"}"},
            {"role": "user", "content": text}
        ],
        temperature=0,
        response_format={"type": "json_object"}
    )
    
    data = json.loads(response.choices[0].message.content)
    results.append(data)

# 写入 CSV（实际项目还会加错误处理、进度条、日志）
with open("orders.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["order_id", "name", "item", "quantity", "phone"])
    writer.writeheader()
    writer.writerows(results)

print("提取完成，结果保存在 orders.csv")
```

**哪几行是必须的**：
- ✅ 必须：`OpenAI()`、`chat.completions.create()`、`messages`、`model`
- ✅ 必须（结构化场景）：`temperature=0`、`response_format`
- ⚠️ 可选但强烈建议：`max_tokens`（防止超支）、错误处理（见第 6 节）

**实际项目里通常会配什么**（进阶，可先跳过）：
- **错误处理**：捕获 `APIError`、`RateLimitError`（429 限流）、`Timeout`
- **重试机制**：失败自动重试 3 次，指数退避
- **成本监控**：记录每次调用消耗的 token，统计总花费
- **并发控制**：用 `httpx` + `asyncio` 并行调用（但注意 RPM 限额）

---

## 第 6 节：常见坑与易错点

### 坑 1：API Key 泄露

**现象**：代码里直接写 `api_key="sk-proj-..."` 并提交 Git，Key 被盗用。

**原因**：API Key 是明文密钥，泄露后任何人都能用你的账号调用（产生费用）。

**解决**：
1. 用环境变量：`export OPENAI_API_KEY="..."`，代码里用 `os.getenv()`
2. 或用 `.env` 文件 + `python-dotenv` 库，并把 `.env` 加入 `.gitignore`

---

### 坑 2：忘记设 temperature=0，每次输出不一样

**现象**：同样的输入，每次返回的 JSON 字段顺序、值不同，导致解析失败。

**原因**：默认 `temperature=1`，模型会随机采样，输出不确定。

**解决**：提取、分类等任务**必须设 temperature=0**，保证输出稳定。

---

### 坑 3：返回的不是合法 JSON

**现象**：即使设了 `response_format={"type": "json_object"}`，`json.loads()` 仍报错。

**原因**：
- prompt 里没明确说"只返回 JSON，不要额外文字"
- 或模型理解错了格式（比如返回了 JSON 数组而你期望对象）

**解决**：
1. system prompt 里加一句"只返回 JSON，不要任何解释或前缀"
2. 调用后先打印 `response.choices[0].message.content` 检查原始输出
3. 加 try-except 捕获 `json.JSONDecodeError`，记录失败案例

---

### 坑 4：token 超限导致截断

**现象**：回复到一半突然停止，JSON 不完整。

**原因**：`max_tokens` 设太小，或输入 + 输出总长度超过模型上下文窗口（如 gpt-3.5-turbo 是 4k tokens）。

**解决**：
1. 给 `max_tokens` 留足够余量（如输出预期 200 token，设 300）
2. 检查 `response.choices[0].finish_reason`，如果是 `length` 表示被截断
3. 输入文本太长时，先用 `tiktoken` 统计 token 数，必要时分段处理

---

### 坑 5：429 错误（Rate Limit Exceeded）

**现象**：循环调用时报 `RateLimitError: You exceeded your RPM quota`。

**原因**：免费/低级账户有 RPM（每分钟请求数）限制，如 3 RPM。

**解决**：
1. 加 `time.sleep(20)` 控制频率（粗暴但有效）
2. 或用 `tenacity` 库做指数退避重试
3. 升级到付费账户提高限额

---

## 第 7 节：验证你学会了

### 问题 1：用一句话说清 OpenAI API 是什么、解决什么问题

**提示**：回查第 1、2 节。

---

### 问题 2：不看文档，默写一个最小调用示例

**要求**：
- 导入库、初始化客户端
- 发送一条 user 消息："今天天气怎么样"
- 打印回复

**提示**：参考第 4 节示例 1。

---

### 问题 3：变体场景

**任务**：写一个函数 `extract_invoice(text: str) -> dict`，输入发票文本，返回 `{"invoice_no": "...", "amount": 数字, "date": "YYYY-MM-DD"}` 格式的字典。要求输出稳定、格式固定。

**提示**：
- 需要哪些参数确保输出稳定？（第 4 节示例 2）
- 如何强制返回 JSON？（第 4 节示例 3）
- 如何处理 `json.loads()` 可能的报错？（第 6 节坑 3）

**答案位置**：第 4 节示例 2、3 + 第 5 节实际用法模板。
