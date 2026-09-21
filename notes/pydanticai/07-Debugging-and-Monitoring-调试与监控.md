# 07 调试与监控 — 为什么 Agent 必须"看着它跑"

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#debugging-and-monitoring  
> 精读日期：2026-09-21  
> 核心问题：Agent 行为是随机的，必须记录每次决策过程

---

## 一句话本质：Logfire = 行车记录仪

Logfire 的本质就一句话：**在运行期间录下大模型的行为，方便事后排查**。

类比行车记录仪——平时不看，但必须一直开着；出事了回放现场。它额外还能告诉你"油耗"（每次 run 的成本）。

> 认知刷新：如果只抱着"先开着、出问题再看"的想法，你只用到它**被动排查**的一半价值。它还有两个**主动**用途（见下文"实战建议"）——开发期确认"模型理解对了"，以及持续的成本监控（否则等账单产生才后知后觉）。

---

## 为什么传统调试方法对 Agent 无效？

**传统软件**：写死的 if-else 逻辑，读代码就能预测行为。  
**Agent**：模型的决策是**随机**的，而且这种随机性会在 agentic loop 中**逐级放大**：

```
用户提问 → 模型推理（随机） → 调工具 → 观察结果 → 再推理（更随机） → ...
```

**结果**：你**必须看到它实际跑了什么**，而不是猜测"应该"跑什么。

**实际场景举例**：
- 你给 Agent 指令："从文档提取公司名和金额"
- 第一次跑：模型调了 `extract_entities` 工具，成功
- 第二次跑同样的文档：模型直接返回 `[]`，没调工具
- 第三次跑：模型调了两次工具，第一次参数错误重试

**没有 trace，你根本不知道哪次对、哪次错、为什么错。**

---

## 最小化接入：两行代码启用 Logfire

```python
import logfire

logfire.configure()  # 第一次运行会提示登录（或本地模式）
logfire.instrument_pydantic_ai()  # 自动记录所有 Agent 运行
```

**就这样。** 之后每次 `agent.run_sync()` 都会自动生成详细 trace。

### <mark style="background: #BBFABBA6;">Trace 包含什么？</mark>

打开 Logfire Web UI（或你的 OpenTelemetry 后端），每次 run 会显示：

1. **消息链路**：system prompt → user → assistant → tool call → tool result → assistant
2. **工具调用细节**：工具名、参数（JSON）、返回值、耗时
3. **Token 用量**：每次请求的 input/output tokens、累计用量、成本（美元）
4. **错误上下文**：如果失败，完整的调用栈 + 当时的消息历史
5. **延迟分布**：哪个环节慢（模型响应？工具执行？）

**对比传统日志**：
```python
# 传统方式（纯文本日志）
logger.info('Calling tool extract_entities with args: ...')  # 需要手写
logger.info('Tool returned: ...')                           # 需要手写
logger.error('Failed: ...', exc_info=True)                  # 缺上下文

# Logfire 方式
# 什么都不用写，自动记录上面所有信息，还能在 Web UI 里交互式查看
```

---

## 实战场景：调试"为什么模型没调工具？"

**问题**：你定义了一个工具 `search_database`，但模型从不调用它。

**传统排查**（猜测法）：
1. 改工具描述？试试
2. 改 system prompt？再试试
3. 换个模型？还是不行
4. 放弃，手动调工具...

**Logfire 排查**（看 trace）：
1. 打开 trace，看 "Request" tab → 模型收到的 system prompt 里**根本没有这个工具**
2. 原因：你在定义工具时写了 `@agent.tool(enabled=False)`，复制粘贴时忘了删
3. 改成 `enabled=True`，下次 trace 里就能看到工具出现在 prompt 中了

**时间差异**：传统方法可能耗费 1 小时，Logfire 方法只需 2 分钟。

---

## 实战场景：优化成本（发现隐藏的 token 浪费）

**问题**：你的提取任务平均每份文档花费 $0.05，但预算是 $0.02。

**Logfire 排查**：
1. 按 `cost` 降序排列所有 trace，找到最贵的那次（$0.12）
2. 打开它，看 "Messages" tab：
   - 第一次请求：6000 input tokens（正常，文档很长）
   - 第二次请求：**12000 input tokens**（翻倍了！为什么？）
3. 展开第二次请求的 system prompt 部分：
   - 模型把**整个文档的提取结果**塞回 prompt 里了（你用了 `message_history`）
   - 加上原始文档，token 数翻倍
4. 解决：不要传 `message_history`，提取任务是单轮的

**节省成本**：从 $0.05 → $0.02（60% 降低）。

---

## 系统化测试：大模型测试的两层结构

大模型测试和普通单元测试不同——输出是随机的，没法 `assert result == "固定字符串"`。实际分两层，各测各的：

| 层 | 测什么 | 工具 | 特点 |
|----|--------|------|------|
| **流程层** | 你的代码（工具注册、deps 传递、异常处理） | `TestModel` / `FunctionModel` | 免费、毫秒级、每次改代码都跑（CI） |
| **质量层** | prompt + 模型的实际表现 | `pydantic_evals` + 真模型 | 花钱、秒级、只有改 prompt / 换模型才跑 |

**第一层：流程测试（写死行为，不调真模型）**

```python
from pydantic_ai.models.test import TestModel

def test_agent_flow():
    with agent.override(model=TestModel()):
        result = agent.run_sync('任意输入')
    assert result.output is not None  # 只验证"流程跑通了"，不验证内容
```

`TestModel` 自动生成符合 output_type 的假输出并调用所有工具——$0、毫秒级。用来防"改代码把流程改坏"。

**关键边界**：这层**不测模型智商**，只测你的代码逻辑。想精确控制模型返回（比如测"模型返回空 / 工具抛异常时，你的重试逻辑对不对"），用 `FunctionModel` 自定义返回：

```python
from pydantic_ai.models.function import FunctionModel
from pydantic_ai.messages import ModelResponse, TextPart

def fake_model(messages, info):
    return ModelResponse(parts=[TextPart('{"amount": 5000}')])  # 你说了算

with agent.override(model=FunctionModel(fake_model)):
    result = agent.run_sync('提取金额')
assert result.output.amount == 5000
```

**整体工作流**：

```
改 instructions
    ↓
第一层：pytest 跑流程测试（TestModel，秒级，免费）—— 代码没坏？
    ↓
第二层：evals 跑 3-5 个真实用例（真模型，花几分钱）—— 质量没降？
    ↓
打开 Logfire trace 扫一眼 —— 模型行为符合预期？
```

---

## Evals：质量层（回归测试 Agent 质量）

**场景**：你改了 instructions，想知道提取质量有没有下降。

```python
from pydantic_evals import Case, Dataset

# 准备测试用例（真实文档 + 期望输出）
dataset = Dataset(
    name='extraction_quality',
    cases=[
        Case(
            name='invoice_001',
            inputs='Invoice #1234\nAmount: $5000\nVendor: Acme Corp',
            expected_output={'invoice_id': '1234', 'amount': 5000, 'vendor': 'Acme Corp'},
        ),
        Case(
            name='invoice_002',
            inputs='Bill No. 5678 | Total $3000 | From XYZ Ltd',
            expected_output={'invoice_id': '5678', 'amount': 3000, 'vendor': 'XYZ Ltd'},
        ),
        # ... 再加 8-10 个真实案例
    ]
)

# 运行评估
def my_extract_function(text: str):
    return extract_agent.run_sync(text).output

report = dataset.evaluate_sync(my_extract_function)
print(f'准确率: {report.accuracy}')  # 0.85 → 改 instructions 后 → 0.92
```

**工作流**：
1. **建立基准**：用当前 instructions 跑一遍，记录分数（如 85% 准确率）
2. **改 instructions**：优化提示词
3. **跑回归**：再跑一遍，对比分数（92% → 提升了！或 70% → 退步了）
4. **持续迭代**：每次改动都跑一遍，确保不倒退

**与 Logfire 结合**：
- Evals 的每次运行都会自动记录到 Logfire
- 在 Web UI 里可以**对比两次运行的 trace**，看具体哪个 case 变好/变差了

---

## 生产环境监控：关注这三个指标

### 1. 成本趋势（防止突然暴涨）

```python
# Logfire 查询示例（伪代码）
SELECT date, AVG(cost) as avg_cost
FROM traces
WHERE agent_name = 'extract_agent'
GROUP BY date
```

**告警规则**：日均成本 > $100 → 发邮件通知

### 2. 失败率（模型行为异常）

**正常**：失败率 < 5%（偶尔网络抖动）  
**异常**：失败率突然 > 20% → 可能是：
- OpenAI API 降级（查看 trace 里的错误信息）
- instructions 改坏了（对比前后 trace）
- 用户输入格式变了（看 user prompt 内容）

### 3. 工具调用频率（行为漂移检测）

**正常**：`search_database` 工具每次 run 平均调用 1.2 次  
**异常**：突然变成 0.3 次 → 模型不再调工具了（可能是模型升级后行为变了）

**Logfire 查询**：
```
SELECT tool_name, COUNT(*) 
FROM tool_calls 
WHERE timestamp > NOW() - INTERVAL '7 days'
GROUP BY tool_name
```

---

## 不用 Logfire？可以用任何 OpenTelemetry 后端

Pydantic AI 的 instrumentation 基于 **OpenTelemetry**（开放标准），所以 trace 可以发到：

- Datadog
- Jaeger
- Grafana Tempo
- AWS X-Ray
- 任何兼容 OTLP 的后端

**配置方式**：见 [alternative backends 文档](https://pydantic.dev/docs/ai/integrations/logfire/#using-opentelemetry)

**即使用 Logfire SDK**，也可以配置它把数据转发到其他后端（Logfire 只是个方便的包装层）。

---

## 实战建议（Day 5 测试阶段）

### <mark style="background: #BBFABBA6;">开发期（必须）</mark>

```python
import logfire

logfire.configure(send_to_logfire=False)  # 本地模式，不上传云端
logfire.instrument_pydantic_ai()

# 之后所有 trace 存本地，用 Logfire CLI 查看
```

**用处**：
- 每次改 instructions 后跑几个测试，<mark style="background: #BBFABBA6;">看 trace 确认模型理解对了</mark>
- 发现 token 浪费（system prompt 太长、重复传历史等）
- 调试工具不被调用的问题

### <mark style="background: #BBFABBA6;">生产期（推荐）</mark>

```python
logfire.configure()  # 上传到 Logfire 云端（或你的 OTLP 后端）
logfire.instrument_pydantic_ai()

# 配合 Evals 做回归测试
dataset.evaluate_sync(my_function)  # 每次部署前跑
```

**告警配置**：
- 日均成本 > 阈值 → 邮件通知
- 失败率 > 10% → 页面告警
- Evals 准确率 < 基准 → 阻止部署

---

## 常见问题

### Q1：Logfire 免费吗？
**A**：有免费层（每月 1GB trace 数据），够个人项目用。生产环境看规模，可能需要付费或自建 OTLP 后端。

### Q2：会不会泄露敏感数据？
**A**：
- 本地模式（`send_to_logfire=False`）：数据不出本机
- 云端模式：默认会记录工具参数和返回值，包含敏感信息的字段用 `@agent.tool(..., redact_result=True)` 屏蔽

### Q3：我用 Datadog，能集成吗？
**A**：能。Pydantic AI → OpenTelemetry → Datadog APM，配置见[文档](https://pydantic.dev/docs/ai/integrations/logfire/#using-opentelemetry)。

### Q4：不想用可观测性工具，能不能自己打日志？
**A**：可以，但**强烈不推荐**：
- 你需要手动记录每个消息、工具调用、token 数（几百行代码）
- 纯文本日志无法交互式查看（不能点开 trace 逐步回放）
- 出问题时排查效率低 10 倍

---

## 总结：为什么必须用 trace

| 场景         | 没有 trace       | 有 trace          |
| ---------- | -------------- | ---------------- |
| 调试"模型不调工具" | 改代码瞎试 1 小时     | 看 trace 2 分钟定位   |
| 成本优化       | 不知道钱花哪了        | 按 cost 排序找元凶     |
| 回归测试       | 手动跑几个例子"感觉没问题" | Evals 自动跑 + 对比分数 |
| 生产监控       | 失败了才发现         | 实时告警 + 趋势分析      |

**记住**：Agent 的行为是**随机**的，不看 trace 就是在赌博。
