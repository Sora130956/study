# 工具调用（Tool Calling）：让 LLM 调用外部函数

- 目标读者：会调用 OpenAI API，想让 LLM 执行实际操作
- 前置知识：函数定义、JSON Schema、字典操作
- 学习时长：1.5 小时 / 可跳过：第 6 节（坑点）可在遇到时再回查

---

## 第 1 节：它是什么

**Tool Calling（工具调用）** 是让 LLM 判断"需要调用哪个外部函数"并生成参数，你的代码负责执行函数并把结果返回给 LLM，形成闭环。

**类比**：像给助手配了一套工具箱——你问他"今天北京天气怎么样"，他判断出"需要查天气工具"，告诉你"请调用 `get_weather(city='北京')`"，你执行后把结果（"晴天 25℃"）告诉他，他再组织成自然语言回复你。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 不用它会怎样？

如果不用 Tool Calling，想让 LLM"查天气"只能：
1. 让 LLM 生成一段文本 `"调用 get_weather，参数 city=北京"`
2. 你用正则表达式解析这段文本，提取函数名和参数
3. 手动调用函数，拿到结果
4. 把结果拼成 prompt 再发给 LLM

**痛点**：
- LLM 返回格式不稳定（可能是 `get_weather('北京')`、`查询北京天气`、`weather(北京)`）
- 解析逻辑脆弱（一改就坏）
- 无法处理复杂参数（多个参数、嵌套对象）
- 每个工具都要写一套解析代码

### 问题根源

**函数调用意图被隐藏在自然语言里**，而自然语言不是结构化的，解析起来既慢又不可靠。

### 经典应用场景

- 联网查询（搜索、查天气、查股票）
- 操作数据库（查询、插入、更新）
- 调用第三方 API（发邮件、创建订单、调用计算器）
- 多步骤任务（LLM 判断需要哪几步，自动编排工具调用顺序）

---

## 第 3 节：读下去之前，先搞懂这些概念

| 术语 | 大白话解释 | 示例 | 对应正文节 |
|------|-----------|------|-----------|
| **tool** | 你提供给 LLM 的"工具"定义，包含函数名、参数、描述 | `{"type": "function", "function": {...}}` | 第 4 节示例 1 |
| **JSON Schema** | 描述参数结构的标准格式 | `{"type": "object", "properties": {...}}` | 第 4 节示例 1 |
| **tool_call** | LLM 返回的"我要调用这个工具"的指令，包含函数名和参数 | `{"name": "get_weather", "arguments": "..."}` | 第 4 节示例 1 |
| **tool_choice** | 控制 LLM 是否必须调用工具：`auto`（自动判断）、`required`（必须调用）、`none`（禁止调用） | `tool_choice="auto"` | 第 4 节示例 2 |
| **role="tool"** | 把工具执行结果返回给 LLM 时，消息的角色类型 | `{"role": "tool", "content": "结果"}` | 第 4 节示例 1 |

**JSON Schema 原理补充**：
```json
{
  "type": "object",
  "properties": {
    "city": {"type": "string", "description": "城市名称"},
    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
  },
  "required": ["city"]
}
```
- `type: object` 表示参数是一个对象（字典）
- `properties` 列出所有字段及其类型
- `required` 标明哪些字段必填

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：单个工具 + 手动调用循环

**目标**：定义一个"查天气"工具，让 LLM 判断何时调用，执行后返回结果。

```python
import json
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 1. 定义工具：查天气函数（模拟实现）
def get_weather(city: str, unit: str = "celsius") -> str:
    """
    查询指定城市的天气
    这里是模拟实现，实际项目应该调用真实天气 API
    """
    return f"{city}今天晴天，温度 25°{unit[0].upper()}"

# 2. 工具的 JSON Schema 定义（告诉 LLM 这个工具的签名）
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",  # 函数名，必须和实际函数名一致
            "description": "查询指定城市的天气",  # LLM 靠这个判断何时调用
            "parameters": {  # 参数结构（JSON Schema 格式）
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如'北京'、'上海'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位"
                    }
                },
                "required": ["city"]  # city 必填，unit 可选
            }
        }
    }
]

# 3. 第一轮调用：让 LLM 判断是否需要调用工具
messages = [{"role": "user", "content": "北京今天天气怎么样？"}]

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
    tools=tools,  # 传入工具定义
    tool_choice="auto"  # 让 LLM 自动判断是否调用
)

message = response.choices[0].message

# 4. 检查 LLM 是否要调用工具
if message.tool_calls:
    # LLM 返回了工具调用请求
    tool_call = message.tool_calls[0]  # 可能有多个工具调用，这里取第一个
    function_name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)  # 参数是 JSON 字符串，需解析
    
    print(f"LLM 决定调用：{function_name}({arguments})")
    
    # 5. 执行工具（实际调用 Python 函数）
    if function_name == "get_weather":
        result = get_weather(**arguments)  # 解包参数传入
    
    # 6. 把工具执行结果返回给 LLM
    messages.append(message)  # 先把 LLM 的 tool_call 消息加入历史
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,  # 必须带上 ID，让 LLM 知道这是哪次调用的结果
        "name": function_name,
        "content": result
    })
    
    # 7. 第二轮调用：LLM 根据工具结果生成最终回复
    final_response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages
    )
    
    print(final_response.choices[0].message.content)
else:
    # LLM 认为不需要调用工具，直接回复
    print(message.content)
```

**逐段解读**：

1. **定义工具函数**：
   - 普通 Python 函数，参数类型注解（方便生成 Schema）
   - 返回字符串（也可以返回字典、列表，但最终要转成字符串传给 LLM）

2. **tools 定义**：
   - `description`：LLM 靠这个判断"用户问题是否需要这个工具"
   - `parameters`：标准 JSON Schema，LLM 根据它生成参数
   - `required`：必填字段列表

3. **第一轮调用**：
   - `tools=tools`：把工具列表传进去
   - `tool_choice="auto"`：让 LLM 自己判断是否调用

4. **检查 tool_calls**：
   - `message.tool_calls` 存在 → LLM 要调用工具
   - 不存在 → LLM 直接回复（比如用户问"你好"）

5. **解析并执行**：
   - `function.arguments` 是 JSON 字符串，用 `json.loads()` 转成字典
   - 用 `**arguments` 解包传参

6. **返回结果**：
   - `role="tool"` 表示这是工具执行结果
   - `tool_call_id` 必须带上（一次对话可能有多个工具调用）

7. **第二轮调用**：
   - LLM 拿到工具结果后，生成自然语言回复（如"北京今天晴天，温度 25°C"）

**这一步你得到了什么**：完整的"用户提问 → LLM 判断 → 执行工具 → 生成回复"闭环。

---

### 示例 2：多个工具 + 自动路由

**目标**：定义两个工具（查天气 + 计算器），让 LLM 根据问题选择调用哪个。

```python
import json
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 定义两个工具
def get_weather(city: str) -> str:
    return f"{city}今天晴天 25°C"

def calculate(expression: str) -> str:
    """
    计算数学表达式
    注意：eval 有安全风险，生产环境需用 ast.literal_eval 或专门的解析库
    """
    try:
        result = eval(expression)
        return str(result)
    except Exception as e:
        return f"计算错误：{e}"

# 工具定义
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名"}
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "计算数学表达式，如 '3 + 5 * 2'",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "数学表达式"}
                },
                "required": ["expression"]
            }
        }
    }
]

# 工具映射表（避免 if-elif 链）
available_functions = {
    "get_weather": get_weather,
    "calculate": calculate
}

# 测试两个问题
questions = [
    "上海今天天气如何？",
    "帮我算一下 15 * 7 + 23"
]

for question in questions:
    print(f"\n问题：{question}")
    messages = [{"role": "user", "content": question}]
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        tools=tools,
        tool_choice="auto"
    )
    
    message = response.choices[0].message
    
    if message.tool_calls:
        tool_call = message.tool_calls[0]
        function_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        
        print(f"→ 调用工具：{function_name}({arguments})")
        
        # 从映射表取函数并执行
        function_to_call = available_functions[function_name]
        result = function_to_call(**arguments)
        
        messages.append(message)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "name": function_name,
            "content": result
        })
        
        final_response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages
        )
        
        print(f"回复：{final_response.choices[0].message.content}")
    else:
        print(f"回复：{message.content}")
```

**逐段解读**：

- **工具映射表**：
  - 用字典把函数名映射到实际函数对象
  - 避免写 `if function_name == "get_weather": ...` 的长链

- **LLM 自动路由**：
  - 第一个问题提到"天气"，LLM 自动选 `get_weather`
  - 第二个问题是数学计算，LLM 自动选 `calculate`

**这一步你得到了什么**：一个可扩展的多工具系统，新增工具只需往 `tools` 和 `available_functions` 里加。

---

### 示例 3：连续多轮工具调用

**目标**：处理"需要多次调用工具才能回答"的问题。

```python
import json
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def get_weather(city: str) -> str:
    return f"{city}今天晴天 25°C"

def get_population(city: str) -> str:
    populations = {"北京": "2100万", "上海": "2500万", "深圳": "1300万"}
    return populations.get(city, "未知")

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市天气",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_population",
            "description": "查询城市人口",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    }
]

available_functions = {
    "get_weather": get_weather,
    "get_population": get_population
}

# 复杂问题：需要两个工具的结果
messages = [{"role": "user", "content": "北京和上海哪个城市人口更多？今天天气分别如何？"}]

# 循环处理工具调用（直到 LLM 不再要求调用工具）
max_iterations = 5  # 防止死循环
for i in range(max_iterations):
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        tools=tools,
        tool_choice="auto"
    )
    
    message = response.choices[0].message
    
    if not message.tool_calls:
        # LLM 不再调用工具，输出最终答案
        print(message.content)
        break
    
    # 处理所有工具调用（可能一次返回多个）
    messages.append(message)
    for tool_call in message.tool_calls:
        function_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        
        print(f"[轮次 {i+1}] 调用：{function_name}({arguments})")
        
        result = available_functions[function_name](**arguments)
        
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "name": function_name,
            "content": result
        })
```

**逐段解读**：

- **循环调用**：
  - 有些问题需要多次工具调用（比如"对比两个城市"需要查两次）
  - 循环直到 `tool_calls` 为空

- **max_iterations**：
  - 防止 LLM 陷入死循环（比如工具返回结果不符合预期，LLM 一直重试）

- **一次返回多个 tool_calls**：
  - LLM 可能判断"需要同时查北京和上海的天气"
  - 用 `for tool_call in message.tool_calls` 遍历处理

**这一步你得到了什么**：能处理复杂的多步骤任务，LLM 自动编排调用顺序。

---

## 第 5 节：最小可用的实际用法

**生产场景模板**：封装一个通用的工具调用执行器。

```python
import json
from openai import OpenAI
from typing import Callable, Dict
import os

class ToolExecutor:
    def __init__(self, api_key: str):
        self.client = OpenAI(api_key=api_key)
        self.tools = []
        self.functions: Dict[str, Callable] = {}
    
    def register_tool(self, func: Callable, description: str, parameters: dict):
        """注册一个工具"""
        self.functions[func.__name__] = func
        self.tools.append({
            "type": "function",
            "function": {
                "name": func.__name__,
                "description": description,
                "parameters": parameters
            }
        })
    
    def run(self, user_message: str, max_iterations: int = 5) -> str:
        """执行对话，自动处理工具调用"""
        messages = [{"role": "user", "content": user_message}]
        
        for i in range(max_iterations):
            response = self.client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages,
                tools=self.tools,
                tool_choice="auto"
            )
            
            message = response.choices[0].message
            
            if not message.tool_calls:
                return message.content
            
            messages.append(message)
            for tool_call in message.tool_calls:
                func_name = tool_call.function.name
                args = json.loads(tool_call.function.arguments)
                
                # 执行工具
                result = self.functions[func_name](**args)
                
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "name": func_name,
                    "content": str(result)
                })
        
        return "达到最大迭代次数"

# 使用示例
executor = ToolExecutor(api_key=os.getenv("OPENAI_API_KEY"))

# 注册工具
def get_weather(city: str) -> str:
    return f"{city}今天晴天 25°C"

executor.register_tool(
    func=get_weather,
    description="查询城市天气",
    parameters={
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
    }
)

# 执行
answer = executor.run("北京今天天气怎么样？")
print(answer)
```

**哪几行是必须的**：
- ✅ 必须：工具的 `description`（LLM 靠它判断何时调用）
- ✅ 必须：`parameters` 的 JSON Schema
- ✅ 必须：`tool_call_id`（返回结果时必须带上）
- ⚠️ 强烈建议：`max_iterations` 限制（防止死循环）

---

## 第 6 节：常见坑与易错点

### 坑 1：description 写得不清楚，LLM 不调用

**现象**：用户明明问了天气，LLM 却说"我无法查询天气"。

**原因**：`description` 太模糊（如"工具1"），LLM 不知道何时用。

**解决**：写清楚功能和使用场景，如"查询指定城市的实时天气，包括温度、天气状况"。

---

### 坑 2：忘记返回 tool_call_id

**现象**：API 报错 `invalid tool_call_id`。

**原因**：返回工具结果时必须带上 `tool_call_id`，否则 LLM 不知道这是哪次调用的结果。

**解决**：`messages.append({"role": "tool", "tool_call_id": tool_call.id, ...})`。

---

### 坑 3：工具函数报错导致整个流程崩溃

**现象**：工具执行失败，程序退出。

**原因**：没有 try-except 处理工具内部异常。

**解决**：
```python
try:
    result = available_functions[function_name](**arguments)
except Exception as e:
    result = f"工具执行失败：{e}"
```

---

### 坑 4：参数类型不匹配

**现象**：LLM 返回 `"city": "北京"`，但函数签名是 `city: int`，报错。

**原因**：JSON Schema 定义的类型和函数签名不一致。

**解决**：确保 Schema 的 `type` 和 Python 类型注解匹配（`str` ↔ `"string"`，`int` ↔ `"integer"`）。

---

### 坑 5：eval() 安全风险

**现象**：用 `eval()` 执行用户输入的表达式，被注入恶意代码。

**原因**：`eval("__import__('os').system('rm -rf /')")` 可以执行任意代码。

**解决**：
- 用 `ast.literal_eval()`（只支持字面量）
- 或用 `sympy`、`numexpr` 等专门的数学表达式解析库

---

## 第 7 节：验证你学会了

### 问题 1：用一句话说清 Tool Calling 解决什么问题

**提示**：回查第 2 节。

---

### 问题 2：不看文档，默写一个最小工具调用示例

**要求**：
- 定义一个工具 `add(a: int, b: int) -> int`，返回两数之和
- 让 LLM 计算 "3 + 5"
- 打印最终回复

**提示**：参考第 4 节示例 1。

---

### 问题 3：变体场景

**任务**：定义两个工具 `search(query: str)` 和 `summarize(text: str)`，实现"先搜索再总结"的流程：
1. 用户问"Python 是什么"
2. LLM 调用 `search("Python")`
3. 你返回一段维基百科文本
4. LLM 再调用 `summarize(text=...)`
5. 输出摘要

**提示**：
- 如何定义多个工具？（`tools` 列表放两个工具）
- 如何循环处理多次调用？（第 4 节示例 3）
- 如何模拟搜索结果？（返回固定字符串即可）
