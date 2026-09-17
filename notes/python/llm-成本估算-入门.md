# LLM API 成本估算入门：算清楚每次调用花多少钱

- **目标读者**：会 Python 基础，但没接触过 LLM API 调用和成本计算
- **前置知识**：Python 基础语法、字典和列表操作、函数定义
- **学习时长**：约 40 分钟
- **使用场景**：Upwork 接单时给客户报价，需要估算 API 成本
- **可跳过的节**：第 6 节（常见坑）可先跳过，遇到问题再回查

---

## 第 1 节：它是什么

**一句话定义**：成本估算函数是一个能在调用 LLM API 之前，根据输入内容算出「这次调用大概要花多少钱」的工具函数。

**生活化类比**：就像你去餐厅点菜前看菜单算账——还没吃就知道大概花多少钱。成本估算函数就是给 LLM API 调用算"菜单价"，让你在发送请求前心里有数，避免「点完才发现账单吓人」的情况。

---

## 第 2 节：它解决什么问题

### 不用它会怎样？

假设你接了一个 Upwork 项目：客户要处理 10,000 张发票，需要用 LLM 提取信息。客户问你「API 成本大概多少？」

**朴素做法**：拍脑袋说「不贵」或者「几十美元吧」，然后希望自己猜对。

**痛点**：
1. **报价时心里没底**：报低了自己贴钱，报高了客户不接单
2. **批量处理前不敢跑**：怕一跑就烧掉几百美元，但不跑又不知道要花多少
3. **无法做产品定价**：如果做 SaaS 工具，不知道单次成本就没法定免费额度和会员价

**问题根源**：LLM API 按 token 计费，不算清楚输入输出有多少 token，就无法知道成本。

### 它的经典应用场景

1. **接单报价**：客户问「处理 1 万条数据要多少钱」时，能给出准确估算
2. **批量处理前预算**：跑 1000 个文件之前，先抽样算一下总成本
3. **产品定价**：做 SaaS 工具时，根据单次成本设计免费额度和付费档位
4. **防护拦截**：用户输入太长时，发送前判断是否超预算，提示截断

---

## 第 3 节：读下去之前，先搞懂这些概念

| 术语 | 大白话解释 | 最小示例 | 对应正文 |
|------|-----------|---------|---------|
| **token** | LLM 处理文本的最小单位，中文大约 1 个字 ≈ 1-1.5 token，英文 1 个词 ≈ 0.75 token | `"你好"` ≈ 2 tokens<br>`"hello"` ≈ 1 token | 第 4 节示例 1 |
| **tiktoken** | OpenAI 提供的 token 计数工具库，能在发送请求前算出文本有多少 token | `enc.encode("你好")` → `[123, 456]` 长度为 2 | 第 4 节示例 1 |
| **messages** | LLM API 的输入格式，是一个列表，每条消息包含 `role` 和 `content` | `[{"role": "user", "content": "你好"}]` | 第 4 节示例 2 |
| **usage** | API 返回的实际 token 使用量，包含 `prompt_tokens`（输入）和 `completion_tokens`（输出） | `{"prompt_tokens": 10, "completion_tokens": 20}` | 第 5 节 |
| **价格表** | 每个模型每百万 token 的价格，输入和输出通常不同 | DeepSeek: 输入 ¥1/百万，输出 ¥2/百万 | 第 4 节示例 3 |

**术语速查**：如果你已经知道 LLM API 的 messages 结构和 token 概念，可以直接跳到第 4 节看代码。

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本 —— 算一段文本有多少 token

**目标**：学会用 tiktoken 把文本转成 token 数量。

```python
import tiktoken

# 获取 OpenAI 的 cl100k_base 编码器（GPT-3.5/GPT-4 使用的编码器）
enc = tiktoken.get_encoding("cl100k_base")

# 要计算的文本
text = "请帮我从这张发票中提取金额和日期"

# 编码成 token 列表
tokens = enc.encode(text)

# 打印 token 数量
print(f"文本: {text}")
print(f"Token 数量: {len(tokens)}")  # 输出大约 15-20 个 token
```

**逐段解读**：

1. **`tiktoken.get_encoding("cl100k_base")`**：获取编码器对象。`cl100k_base` 是 OpenAI GPT-3.5/GPT-4 使用的编码规则，不同模型可能用不同编码器。
2. **`enc.encode(text)`**：把文本转成 token ID 列表。比如 `"你好"` 可能被编码成 `[12345, 67890]`，长度为 2。
3. **`len(tokens)`**：列表长度就是 token 数量，这个数字决定了 API 调用的成本。

**原理展开**：为什么要"编码"？
- LLM 不直接处理文字，而是处理数字（token ID）
- 编码器把 `"你好"` 拆成 token，每个 token 对应一个 ID
- 中文通常 1 个字 = 1-1.5 token，英文 1 个词 = 0.75 token 左右

**这一步你得到了什么**：学会了把任意文本转成 token 数量，这是成本估算的基础。

---

### 示例 2：加真实需求 —— 算一次 LLM 对话的输入 token

**目标**：LLM API 的输入是 `messages` 列表，学会算整个对话的 token 数。

```python
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

def count_messages_tokens(messages: list[dict]) -> int:
    """
    计算 messages 列表的总 token 数
    
    Args:
        messages: LLM API 的输入格式，如 [{"role": "user", "content": "你好"}]
    
    Returns:
        总 token 数（包含消息结构的开销）
    """
    total_tokens = 3  # 每次对话有 3 个固定的 special token（开始/结束标记）
    
    for message in messages:
        total_tokens += 4  # 每条消息有 4 个结构 token（role 标记等）
        
        # 计算 content 的 token 数
        content = message.get("content") or ""
        total_tokens += len(enc.encode(content))
    
    return total_tokens


# 实际使用
messages = [
    {"role": "system", "content": "你是一个发票信息提取助手"},
    {"role": "user", "content": "请从这张发票中提取金额：$1,234.56，日期：2024-03-15"}
]

token_count = count_messages_tokens(messages)
print(f"这次对话的输入 token 数: {token_count}")  # 输出约 30-40
```

**逐段解读**：

1. **为什么有固定的 3 和 4**：OpenAI 的 API 在处理 messages 时，会添加一些结构标记（如 `<|im_start|>`, `<|im_end|>`），这些标记也占 token。3 是对话级别的，4 是每条消息的。
2. **`message.get("content") or ""`**：安全地获取 content，如果没有就用空字符串（避免 content 为 None 时报错）。
3. **累加逻辑**：总 token = 对话固定开销 + 每条消息开销 + 每条消息内容的 token。

**和示例 1 的区别**：示例 1 只算纯文本，示例 2 算的是完整 API 输入，包含了 role/content 结构的开销。

**这一步你得到了什么**：能准确计算一次 LLM 调用的输入 token，这是估算成本的核心数据。

---

### 示例 3：贴近项目 —— 完整的成本估算函数

**目标**：结合输入 token、输出 token、价格表，算出一次调用的成本。

```python
import tiktoken
from typing import Optional

enc = tiktoken.get_encoding("cl100k_base")

# 价格表（单位：美元/百万 token）
PRICE_TABLE = {
    "gpt-3.5-turbo": {"input": 0.5, "output": 1.5},      # OpenAI
    "gpt-4": {"input": 30.0, "output": 60.0},             # OpenAI
    "deepseek-chat": {"input": 0.14, "output": 0.28},    # DeepSeek（约 ¥1/百万 ≈ $0.14）
}


def count_messages_tokens(messages: list[dict]) -> int:
    """计算 messages 的输入 token 数"""
    total_tokens = 3
    for message in messages:
        total_tokens += 4
        content = message.get("content") or ""
        total_tokens += len(enc.encode(content))
    return total_tokens


def estimate_cost(
    messages: list[dict],
    estimated_output_tokens: int,
    model: str = "deepseek-chat"
) -> dict:
    """
    估算一次 LLM API 调用的成本
    
    Args:
        messages: 输入的对话列表
        estimated_output_tokens: 预估的输出 token 数（调用前不存在，只能估）
        model: 模型名称，必须在 PRICE_TABLE 中
    
    Returns:
        包含详细信息的字典：
        {
            "input_tokens": 输入 token 数,
            "estimated_output_tokens": 预估输出 token 数,
            "cost_usd": 预估成本（美元）,
            "model": 模型名称
        }
    """
    if model not in PRICE_TABLE:
        raise ValueError(f"模型 {model} 不在价格表中，可用模型：{list(PRICE_TABLE.keys())}")
    
    # 计算输入 token
    input_tokens = count_messages_tokens(messages)
    
    # 获取价格
    price_in = PRICE_TABLE[model]["input"]
    price_out = PRICE_TABLE[model]["output"]
    
    # 计算成本（token 数 × 价格 / 1,000,000）
    cost_usd = (input_tokens * price_in + estimated_output_tokens * price_out) / 1_000_000
    
    return {
        "input_tokens": input_tokens,
        "estimated_output_tokens": estimated_output_tokens,
        "cost_usd": cost_usd,
        "model": model
    }


# 实际使用示例
messages = [
    {"role": "system", "content": "你是一个发票信息提取助手"},
    {"role": "user", "content": "请从这张发票中提取金额：$1,234.56，日期：2024-03-15"}
]

# 输出 token 估算：抽取类任务输出通常很短，估 200 token
result = estimate_cost(messages, estimated_output_tokens=200, model="deepseek-chat")

print(f"模型: {result['model']}")
print(f"输入 token: {result['input_tokens']}")
print(f"预估输出 token: {result['estimated_output_tokens']}")
print(f"预估成本: ${result['cost_usd']:.6f} 美元")
```

**逐段解读**：

1. **PRICE_TABLE 结构**：字典的字典，第一层键是模型名，第二层键是 `input`/`output`，值是每百万 token 的价格（美元）。
2. **`estimated_output_tokens` 参数**：输出是模型生成的，调用前不存在，只能靠估算。后面第 5 节会讲怎么估。
3. **成本计算公式**：`(输入token × 输入价格 + 输出token × 输出价格) / 1,000,000`。除以 1,000,000 是因为价格表单位是「每百万 token」。
4. **返回字典而非单个数字**：方便调用方查看详细信息（输入多少、输出多少、用的哪个模型），便于调试和记录。

**关键设计点**：
- **价格表可配置**：不同模型价格不同，集中管理方便更新
- **输入能算准，输出只能估**：输入在发送前就存在，能精确计算；输出是模型生成的，只能估
- **返回详细信息**：不只返回总成本，还返回中间数据，方便验证和调试

**这一步你得到了什么**：一个可以直接用在项目里的成本估算函数，能在发送请求前算出「这次调用大概花多少钱」。

---

## 第 5 节：实际项目中的用法 —— Upwork 接单报价场景

### 场景：客户问「处理 10,000 张发票要多少钱」

**完整流程**：

```python
# ========== 第 1 步：抽样实测，获取真实的输出 token 数 ==========
# 先跑 10-20 个样本，获取实际的 usage 数据

from openai import OpenAI

client = OpenAI(api_key="your-api-key", base_url="https://api.deepseek.com")

# 假设这是客户提供的样本发票文本
sample_invoice = """
发票号：INV-2024-001
日期：2024-03-15
金额：$1,234.56
"""

messages = [
    {"role": "system", "content": "你是发票信息提取助手，输出 JSON 格式"},
    {"role": "user", "content": f"提取发票信息：{sample_invoice}"}
]

# 实际调用一次，获取 usage
response = client.chat.completions.create(
    model="deepseek-chat",
    messages=messages
)

usage = response.usage
print(f"实测 usage: {usage}")
# 输出示例：Usage(prompt_tokens=45, completion_tokens=180, total_tokens=225)

# ========== 第 2 步：用实测数据估算批量成本 ==========
avg_input_tokens = usage.prompt_tokens      # 45
avg_output_tokens = usage.completion_tokens  # 180

# 估算 10,000 次调用的总成本
total_input_tokens = avg_input_tokens * 10_000
total_output_tokens = avg_output_tokens * 10_000

# DeepSeek 价格（美元/百万 token）
price_in = 0.14
price_out = 0.28

total_cost_usd = (total_input_tokens * price_in + total_output_tokens * price_out) / 1_000_000

print(f"\n=== 给客户的报价估算 ===")
print(f"处理数量: 10,000 张发票")
print(f"单次输入: {avg_input_tokens} tokens")
print(f"单次输出: {avg_output_tokens} tokens")
print(f"API 成本: ${total_cost_usd:.2f} 美元")
print(f"建议报价: ${total_cost_usd * 1.5:.2f} 美元（含 50% 利润）")
```

**关键代码说明**：

1. **`response.usage`（必须）**：这是 API 返回的实际 token 使用量，是计费的准确数字。`prompt_tokens` 是输入，`completion_tokens` 是输出。
2. **为什么要抽样实测**：输出 token 是模型生成的，事前无法精确知道。抽样跑几次，取平均值，得到的估算最准确。
3. **× 1.5 加利润**：API 成本只是项目成本的一部分，还有开发时间、维护成本，所以报价通常是 API 成本的 1.5-2 倍。

**这段代码在项目中的位置**：
- **必须**：`response.usage` 的解析（获取实际成本）
- **必须**：价格表和成本计算公式（PRICE_TABLE 和 `/ 1_000_000`）
- **可选**：抽样估算逻辑（如果客户提供样本数据才用）
- **进阶**（可先跳过）：多模型价格对比、成本监控日志、超预算告警

### 成本算谁的？三种合作模式

算出成本后，下一个问题是：这钱谁出？看项目形态决定。

**模式 1：客户提供 API key（长期项目首选）**

- **适用**：长期维护的定制系统、数据敏感项目（数据不过你账号）、客户已有企业 LLM 账号
- **操作**：客户给你 key，你写代码用他们的配额，账单走他们账户
- **你的收益**：零成本风险，只收开发费 + 维护费
- **注意**：合同里明确「API 成本由客户自行承担」，避免后期扯皮

**模式 2：你垫付，打包在报价里（Upwork 小项目常见）**

- **适用**：一次性交付项目（处理一批数据就完事）、客户不懂技术没有 API 账号、你想控制技术栈选择权
- **操作**：用自己的 key 跑，成本算进报价。报价 = API 成本估算 × 1.5~2.0
- **关键**：合同写死「包含 X 次调用」或「处理 Y 条数据」，超出部分另算——客户说 1 万条、干到一半变 10 万条是常见事故
- **风险**：模型涨价你亏钱，可在报价里加「API 成本浮动条款」

**模式 3：按用量分摊（SaaS / API 服务）**

- **适用**：你做 SaaS 工具按月/按次收费，或你包装一层自己的接口给客户
- **操作**：用自己的 key 统一管理所有客户调用，定价 = API 成本 × 2~5 倍（倍数取决于附加价值：数据处理、UI、运维）
- **例子**：你做了发票识别 API，后端调 DeepSeek 成本 $0.001/次，收客户 $0.005/次——客户不用管 key、模型选型、重试，只管调接口

**按项目金额选模式**：

| 项目类型 | 推荐方案 | 理由 |
|---------|---------|------|
| < $500 一次性项目 | 你垫付，打包报价 | 客户大概率不懂 API，要 key 也搞不定 |
| $500~$2000 定制开发 | 客户提供 key（首选） | 成本可控，你只收开发费 |
| > $2000 或长期维护 | 客户提供 key，或成本+ | 大项目必须拆清成本归属 |
| SaaS 工具/API 服务 | 你垫付 + 按量收费 | 这是商业模式，不是成本 |

**Proposal 写法示例**：

客户提供 key 的场景——

> **API Cost:** This project requires OpenAI/DeepSeek API access. Based on sample data, estimated cost is **$X per 10K records**. You'll provide your own API key, and the cost will be billed directly to your account. I'll help you set up the key and monitor usage.

你垫付的场景——

> **Pricing includes API costs:** My quote of $XXX covers development + API usage for processing up to **10,000 invoices**. Additional batches: $XX per 1,000 records.

**一句话总结**：客户有技术能力 + 长期项目 → 让客户提供 key；Upwork 小单 + 客户小白 → 你垫付打包报价，但 scope 写死数量上限；你做产品 → 你的 key 你的成本，定价时成本 × 3~5 倍覆盖运营。

---

## 第 6 节：常见坑与易错点

### 坑 1：用错编码器导致估算不准

**现象**：用 tiktoken 算出的 token 数，和 API 返回的 `usage.prompt_tokens` 相差很大（超过 10%）。

**原因**：不同模型用不同编码器。`cl100k_base` 只适用于 OpenAI 的 GPT-3.5/GPT-4，DeepSeek、通义千问等国产模型用自己的编码器，用 `cl100k_base` 算只是近似值。

**解决**：
- OpenAI 模型：用 `tiktoken.encoding_for_model("gpt-3.5-turbo")` 自动匹配
- 其他模型：tiktoken 算出来的只是估算，实际计费以 `response.usage` 为准。估算时加 10%-20% 余量

```python
# 正确做法：根据模型选编码器
if model.startswith("gpt"):
    enc = tiktoken.encoding_for_model(model)
else:
    enc = tiktoken.get_encoding("cl100k_base")  # 其他模型只能近似
```

### 坑 2：流式调用拿不到 usage

**现象**：用 `stream=True` 时，response 里没有 `usage` 字段。

**原因**：<mark style="background: #BBFABBA6;">流式调用默认不返回 usage，需要显式开启。</mark>

**解决**：<mark style="background: #BBFABBA6;">加上 `stream_options={"include_usage": True}`</mark>

```python
response = client.chat.completions.create(
    model="deepseek-chat",
    messages=messages,
    stream=True,
    stream_options={"include_usage": True}  # 必须加这行
)

for chunk in response:
    if chunk.usage:  # usage 在最后一个 chunk 里
        print(f"实际 token 使用: {chunk.usage}")
```

### 坑 3：<mark style="background: #BBFABBA6;">忘记输入输出价格不同</mark>

**现象**：成本估算总是偏低。

**原因**：<mark style="background: #BBFABBA6;">几乎所有模型的输出价格都是输入价格的 2-3 倍，但计算时用了同一个价格。</mark>

**解决**：<mark style="background: #BBFABBA6;">价格表必须分 `input` 和 `output`，计算时分别乘。</mark>

```python
# 错误做法
cost = total_tokens * price  # 没区分输入输出

# 正确做法
cost = (input_tokens * price_in + output_tokens * price_out) / 1_000_000
```

### 坑 4：思考模式（reasoning）的隐藏 token 计费

**现象**：<mark style="background: #BBFABBA6;">开启思考模式后，实际成本比估算高出好几倍——最终答案明明只有 200 token，账单上却显示输出 2,000+。</mark>

**原因**：<mark style="background: #BBFABBA6;">模型「思考过程」（推理链）的 token **一律按输出价格计费**，三家厂商口径完全一致，即使这些内容在响应里看不到（或单独放在 `reasoning_content` / `thinking` 字段）。而输出单价通常是输入的 2~4 倍，思考量又往往是答案本身的 5~10 倍，两者叠加导致成本暴涨。
</mark>
**三家对比（2026-09）**：

|         | OpenAI                                         | Claude                                        | DeepSeek                       |
| ------- | ---------------------------------------------- | --------------------------------------------- | ------------------------------ |
| 叫法      | reasoning（GPT-5.x/6 自带，o 系列纯推理）                | Extended Thinking                             | 思考模式 `thinking`                |
| 思考内容可见吗 | 隐藏，不可见                                         | `thinking` block（摘要）                          | `reasoning_content` 字段         |
| 计费      | 计入输出 token                                     | 计入输出 token，且占用 `max_tokens` 上限                | 计入输出 token                     |
| 查看思考用量  | `usage.output_tokens_details.reasoning_tokens` | `usage.output_tokens_details.thinking_tokens` | 混在 `completion_tokens` 里，不单独拆分 |
| 成本控制手段  | `reasoning_effort="low"` 调档                    | 不开启，或设 `budget_tokens` 硬上限                    | `thinking=False` 直接关           |

**解决**：

1. **对估算函数的影响**：思考型模型的 `estimated_output_tokens` 必须按「思考 + 答案」一起估，不能只估答案
2. **抽取类任务直接关思考**：<mark style="background: #BBFABBA6;">发票/文档提取不需要深度推理，关掉后速度快 5~10 倍、成本低 5 倍以上</mark>
3. **需要思考时用预算控制**：Claude 的 `budget_tokens` 是硬上限（注意 `max_tokens` 必须大于它）；OpenAI 调低 `reasoning_effort`；DeepSeek 注意思考模式默认开启，需显式关闭
4. **多轮工具调用的雪球效应**（DeepSeek）：思考模式下做工具调用，后续请求必须回传 `reasoning_content`，这部分又变成下次调用的输入 token

**坑中坑**：DeepSeek 的思考 token 和最终答案混在 `completion_tokens` 里不拆分，你无法直接知道有多少钱花在了「思考」上——对比 OpenAI/Claude 有专门的 details 字段，DeepSeek 最不透明。

### 和直觉相反的点：中文比英文费 token

<mark style="background: #BBFABBA6;">直觉上觉得中文更简洁（1 个字表达的信息量 > 1 个英文词），但在 token 计费上：</mark>
- <mark style="background: #BBFABBA6;">中文：1 个字 ≈ 1-1.5 token</mark>
- <mark style="background: #BBFABBA6;">英文：1 个词 ≈ 0.75 token</mark>

<mark style="background: #BBFABBA6;">所以同样的语义，中文 prompt 通常比英文贵 20%-30%。如果成本敏感，可以考虑用英文写 system prompt。</mark>

---

## 第 7 节：验证你学会了

### 问题 1：复述核心概念

不看文档，用自己的话回答：
- 成本估算函数是什么？
- 它解决什么问题？
- 为什么调用前能算输入 token，但输出 token 只能估？

**答案位置**：第 1 节、第 2 节、第 4 节示例 3 的「关键设计点」

### 问题 2：默写最小版本

不看代码，尝试写出：
1. 用 tiktoken 计算一段文本的 token 数（示例 1）
2. 一个最简单的成本计算公式（只需要公式，不用完整函数）

**答案位置**：第 4 节示例 1、示例 3 的成本计算公式部分

### 问题 3：实战变体

客户给了一个新需求：「我要处理 5,000 个客户咨询对话，每个对话平均 3 轮（用户-助手-用户-助手-用户-助手），你算一下 API 成本」。

要求：
1. 构造一个 3 轮对话的 messages 示例
2. 用 `count_messages_tokens` 算输入 token
3. 估算输出 token（提示：对话类任务输出通常是输入的 0.5-1 倍）
4. 算出单次对话成本，再 × 5,000

**提示**：
- messages 列表会有 6 条（3 轮 × 2 条/轮）
- 输出 token 估算可以用「输入 token × 0.8」作为经验值
- 答案位置：第 4 节示例 2（messages 结构）+ 示例 3（成本计算）+ 第 5 节（批量外推）
