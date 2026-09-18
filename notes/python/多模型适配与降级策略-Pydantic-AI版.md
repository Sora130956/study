# 多模型适配与降级策略 - Pydantic AI 版

## 1. 它是什么（一句话 + 一句类比）

**一句话定义**：使用 Pydantic AI 的统一 `Agent` 接口同时支持多个 LLM 提供商（OpenAI / Anthropic / DeepSeek 等），并在主模型失败时自动切换到备用模型。

**一句类比**：就像手机的双卡双待功能 — 主号欠费了自动切换到副号，用户感知不到中断。

---

## 2. 它解决什么问题 / 为什么需要它

### 痛点 1：手写适配器代码太繁琐
不同 LLM 提供商的 SDK 接口各不相同：
- OpenAI 用 `client.chat.completions.create()`
- Anthropic 用 `client.messages.create()`
- DeepSeek 又是另一套参数格式

每次切换模型都要重写一堆代码，维护成本高。

### 痛点 2：生产环境需要高可用保障
- OpenAI 可能限流或宕机
- 成本控制：便宜模型优先，贵的做备份
- 客户要求：某些客户指定只用国产模型

手写 try-catch 嵌套既难看又容易出 bug。

### Pydantic AI 如何解决
- **统一接口**：所有模型都通过 `Agent(model_name)` 调用，一行代码切换
- **配置驱动**：通过配置文件或环境变量管理多个模型
- **易封装**：基于 Agent 封装降级逻辑只需十几行代码

---

## 3. 读下去之前，先搞懂这些概念

### 3.1 基础概念

**多模型适配（Multi-Model Adapter）**  
<mark style="background: #BBFABBA6;">用同一套代码调用不同的 LLM，切换模型时只改配置不改代码。</mark>

**降级策略（Fallback Strategy）**  
<mark style="background: #BBFABBA6;">主模型失败后按预设顺序尝试备用模型，直到成功或全部失败。</mark>

### 3.2 Pydantic AI 核心概念

**`Agent`**  
Pydantic AI 的核心类，封装了模型调用、提示词管理、结果校验等功能。

```python
from pydantic_ai import Agent

# 创建一个使用 GPT-4 的 Agent
agent = Agent('openai:gpt-4')
```

**模型标识符格式**  
Pydantic AI 使用统一的字符串标识符：
- OpenAI: `'openai:gpt-4'`、`'openai:gpt-4o-mini'`
- Anthropic: `'anthropic:claude-3-5-sonnet-20241022'`
- DeepSeek: `'deepseek:deepseek-chat'`
- Gemini: `'gemini:gemini-1.5-flash'`

**`run_sync()` 和 `run()`**  
- `run_sync()`：<mark style="background: #BBFABBA6;">同步调用，适合脚本和 CLI</mark>
- `run()`：<mark style="background: #BBFABBA6;">异步调用，适合 Web 服务和高并发场景</mark>

**`model_settings`**  
<mark style="background: #BBFABBA6;">传递模型参数（temperature、max_tokens 等）</mark>：

```python
agent = Agent(
    'openai:gpt-4',
    model_settings={'temperature': 0.7, 'max_tokens': 1000}
)
```

---

## 4. 从最简单的代码示例开始

### 示例 1：最小可运行版本（多模型切换）

```python
from pydantic_ai import Agent

# 创建三个不同模型的 Agent
agent_gpt4 = Agent('openai:gpt-4')
agent_claude = Agent('anthropic:claude-3-5-sonnet-20241022')
agent_deepseek = Agent('deepseek:deepseek-chat')

# 同一个提示词，三个模型都能用
prompt = '用一句话解释什么是量子纠缠'

result_gpt4 = agent_gpt4.run_sync(prompt)
result_claude = agent_claude.run_sync(prompt)
result_deepseek = agent_deepseek.run_sync(prompt)

print(f'GPT-4: {result_gpt4.data}')
print(f'Claude: {result_claude.data}')
print(f'DeepSeek: {result_deepseek.data}')
```

**对比原生 API**：如果用原生 SDK，需要分别导入 `openai`、`anthropic`、`deepseek_sdk`，写三套不同的调用代码。Pydantic AI 只需改一个字符串。

---

### 示例 2：配置驱动的模型管理

```python
import os
from pydantic_ai import Agent

# 通过环境变量或配置文件选择模型
model_name = os.getenv('LLM_MODEL', 'openai:gpt-4o-mini')  # 默认用便宜的 mini
agent = Agent(model_name)

prompt = '总结这段文本的核心观点'
result = agent.run_sync(prompt)
print(result.data)
```

**`.env` 文件示例**：
```env
# 开发环境用便宜模型
LLM_MODEL=openai:gpt-4o-mini

# 生产环境用高质量模型
# LLM_MODEL=anthropic:claude-3-5-sonnet-20241022

# API Keys
OPENAI_API_KEY=sk-xxx
ANTHROPIC_API_KEY=sk-ant-xxx
DEEPSEEK_API_KEY=sk-xxx
```

**使用方法**：
```python
from dotenv import load_dotenv
load_dotenv()  # 自动加载 .env 文件
```

---

### 示例 3：带降级的 AgentManager

```python
from pydantic_ai import Agent
from typing import List

class AgentManager:
    """支持降级的多模型管理器"""
    
    def __init__(self, model_names: List[str]):
        """
        初始化多个 Agent
        
        Args:
            model_names: 模型列表，按优先级排序
                        例如 ['openai:gpt-4', 'deepseek:deepseek-chat']
        """
        self.agents = [Agent(name) for name in model_names]
        self.model_names = model_names
    
    def run_with_fallback(self, prompt: str) -> str:
        """
        依次尝试每个模型，直到成功
        
        Returns:
            成功的模型返回结果
        
        Raises:
            RuntimeError: 所有模型都失败
        """
        last_error = None
        
        for i, agent in enumerate(self.agents):
            try:
                print(f'尝试模型: {self.model_names[i]}')
                result = agent.run_sync(prompt)
                print(f'✅ {self.model_names[i]} 成功')
                return result.data
            except Exception as e:
                print(f'❌ {self.model_names[i]} 失败: {e}')
                last_error = e
                continue
        
        raise RuntimeError(f'所有模型都失败了，最后的错误: {last_error}')


# 使用示例
if __name__ == '__main__':
    # 优先用 GPT-4，失败了降级到 DeepSeek
    manager = AgentManager([
        'openai:gpt-4',
        'deepseek:deepseek-chat'
    ])
    
    prompt = '写一首关于代码的打油诗'
    result = manager.run_with_fallback(prompt)
    print(f'\n最终结果:\n{result}')
```

**运行效果**：
```
尝试模型: openai:gpt-4
❌ openai:gpt-4 失败: Rate limit exceeded
尝试模型: deepseek:deepseek-chat
✅ deepseek:deepseek-chat 成功

最终结果:
代码千万行，注释第一行
命名不规范，同事两行泪
```

---

## 5. 最小可用的实际用法（生产场景模板）

### 5.1 完整项目结构

```
my_ai_app/
├── .env                    # 环境变量配置
├── config.yaml             # 模型配置文件
├── requirements.txt        # 依赖列表
├── src/
│   ├── __init__.py
│   ├── agent_manager.py    # 核心：多模型管理器
│   ├── models.py           # 数据模型定义
│   └── utils.py            # 工具函数
├── cli.py                  # CLI 入口
└── tests/
    └── test_agent_manager.py
```

### 5.2 配置文件 `config.yaml`

```yaml
# 模型优先级配置（按顺序降级）
models:
  primary: openai:gpt-4o-mini
  fallback:
    - deepseek:deepseek-chat
    - anthropic:claude-3-5-haiku-20241022

# 模型参数配置
model_settings:
  openai:gpt-4o-mini:
    temperature: 0.7
    max_tokens: 2000
  deepseek:deepseek-chat:
    temperature: 0.8
    max_tokens: 1500
  anthropic:claude-3-5-haiku-20241022:
    temperature: 0.7
    max_tokens: 2000

# 重试配置
retry:
  max_attempts: 3
  backoff_seconds: 2
```

### 5.3 核心代码 `src/agent_manager.py`

```python
import yaml
import time
import logging
from pathlib import Path
from typing import List, Dict, Any
from pydantic_ai import Agent

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class ProductionAgentManager:
    """生产级多模型管理器"""
    
    def __init__(self, config_path: str = 'config.yaml'):
        """从配置文件初始化"""
        self.config = self._load_config(config_path)
        self.agents = self._build_agents()
        self.retry_config = self.config.get('retry', )
    
    def _load_config(self, config_path: str) -> Dict[str, Any]:
        """加载 YAML 配置"""
        with open(config_path, 'r', encoding='utf-8') as f:
            return yaml.safe_load(f)
    
    def _build_agents(self) -> List[tuple]:
        """构建 Agent 列表（带模型名称）"""
        models_config = self.config['models']
        model_settings = self.config.get('model_settings', {})
        
        # 主模型 + 降级模型
        all_models = [models_config['primary']] + models_config.get('fallback', [])
        
        agents = []
        for model_name in all_models:
            settings = model_settings.get(model_name, {})
            agent = Agent(model_name, model_settings=settings)
            agents.append((model_name, agent))
            logger.info(f'初始化模型: {model_name}, 参数: {settings}')
        
        return agents
    
    def run_with_fallback(self, prompt: str, system_prompt: str = None) -> Dict[str, Any]:
        """
        带降级和重试的执行
        
        Returns:
            {
                'success': bool,
                'model_used': str,
                'result': str,
                'attempts': int
            }
        """
        max_attempts = self.retry_config.get('max_attempts', 3)
        backoff = self.retry_config.get('backoff_seconds', 2)
        
        for model_name, agent in self.agents:
            for attempt in range(1, max_attempts + 1):
                try:
                    logger.info(f'[{model_name}] 尝试 {attempt}/{max_attempts}')
                    
                    # 如果提供了 system_prompt，需要重新创建 Agent
                    if system_prompt:
                        agent_with_system = Agent(
                            model_name,
                            system_prompt=system_prompt,
                            model_settings=agent._model_settings
                        )
                        result = agent_with_system.run_sync(prompt)
                    else:
                        result = agent.run_sync(prompt)
                    
                    logger.info(f'✅ [{model_name}] 成功')
                    return {
                        'success': True,
                        'model_used': model_name,
                        'result': result.data,
                        'attempts': attempt
                    }
                
                except Exception as e:
                    logger.warning(f'❌ [{model_name}] 尝试 {attempt} 失败: {e}')
                    
                    if attempt < max_attempts:
                        time.sleep(backoff * attempt)  # 指数退避
                        continue
                    else:
                        logger.error(f'[{model_name}] 达到最大重试次数')
                        break
        
        # 所有模型都失败
        return {
            'success': False,
            'model_used': None,
            'result': None,
            'attempts': max_attempts * len(self.agents)
        }


# 使用示例
if __name__ == '__main__':
    manager = ProductionAgentManager('config.yaml')
    
    result = manager.run_with_fallback(
        prompt='解释什么是 Kubernetes',
        system_prompt='你是一个云原生技术专家'
    )
    
    if result['success']:
        print(f"成功! 使用模型: {result['model_used']}")
        print(f"尝试次数: {result['attempts']}")
        print(f"结果:\n{result['result']}")
    else:
        print(f"失败! 已尝试 {result['attempts']} 次")
```

### 5.4 CLI 入口 `cli.py`

```python
import click
from src.agent_manager import ProductionAgentManager

@click.command()
@click.argument('prompt')
@click.option('--config', default='config.yaml', help='配置文件路径')
@click.option('--system-prompt', default=None, help='系统提示词')
def main(prompt: str, config: str, system_prompt: str):
    """AI 问答 CLI 工具"""
    manager = ProductionAgentManager(config)
    
    result = manager.run_with_fallback(prompt, system_prompt)
    
    if result['success']:
        click.echo(click.style(f"✅ 成功 | 模型: {result['model_used']}", fg='green'))
        click.echo(result['result'])
    else:
        click.echo(click.style('❌ 所有模型都失败了', fg='red'))
        exit(1)

if __name__ == '__main__':
    main()
```

**使用方法**：
```bash
# 基础用法
python cli.py "什么是机器学习"

# 自定义系统提示词
python cli.py "推荐一本书" --system-prompt="你是一个图书管理员"

# 指定配置文件
python cli.py "今天天气" --config=prod.yaml
```

### 5.5 依赖文件 `requirements.txt`

```txt
pydantic-ai==0.0.14
pydantic==2.10.4
openai==1.57.4
anthropic==0.42.0
python-dotenv==1.0.1
PyYAML==6.0.2
click==8.1.8
```

---

## 6. 常见坑与易错点

### 坑点 1：Pydantic AI 没有内置降级机制

**问题**：
```python
# ❌ 这样写不会自动降级
agent = Agent('openai:gpt-4')
result = agent.run_sync(prompt)  # 失败了就直接抛异常，不会切换模型
```

**解决方案**：
必须自己封装 try-catch 逻辑（参考示例 3 的 `AgentManager`）。

---

### 坑点 2：不同模型的 token 计数方式不同

**问题**：
```python
# GPT-4 和 Claude 对同一段文本的 token 数可能相差 20%
# 会影响成本估算和 max_tokens 设置
```

**解决方案**：
```python
# 为每个模型单独配置 max_tokens
model_settings = {
    'openai:gpt-4': {'max_tokens': 2000},
    'anthropic:claude-3-5-sonnet-20241022': {'max_tokens': 2400},  # Claude 需要多一些
}
```

---

### 坑点 3：API Key 混乱导致的报错

**问题**：
```python
# 环境变量命名不一致
OPENAI_API_KEY=sk-xxx
ANTHROPIC_KEY=sk-ant-xxx  # ❌ 错了，应该是 ANTHROPIC_API_KEY
```

**Pydantic AI 期望的环境变量名**：
- OpenAI: `OPENAI_API_KEY`
- Anthropic: `ANTHROPIC_API_KEY`
- DeepSeek: `DEEPSEEK_API_KEY`
- Gemini: `GEMINI_API_KEY`

**解决方案**：
```python
# 在代码中显式检查
import os

required_keys = {
    'openai:gpt-4': 'OPENAI_API_KEY',
    'anthropic:claude-3-5-sonnet-20241022': 'ANTHROPIC_API_KEY',
}

for model, env_var in required_keys.items():
    if not os.getenv(env_var):
        raise ValueError(f'缺少环境变量: {env_var}')
```

---

### 坑点 4：模型切换时 system_prompt 的兼容性问题

**问题**：
```python
# 某些模型对 system_prompt 的格式要求不同
# GPT-4 支持长篇 system_prompt，但某些小模型会截断
```

**解决方案**：
```python
# 为不同模型准备不同的 system_prompt 版本
system_prompts = {
    'openai:gpt-4': '你是一个专业的 Python 工程师，精通 FastAPI、Django 和异步编程...',  # 长版本
    'deepseek:deepseek-chat': '你是 Python 工程师',  # 短版本
}

model_name = 'deepseek:deepseek-chat'
agent = Agent(model_name, system_prompt=system_prompts[model_name])
```

---

### 坑点 5：降级时的成本控制失效

**问题**：
```python
# 原本用便宜模型，结果降级到贵模型，成本暴涨
manager = AgentManager([
    'openai:gpt-4o-mini',  # $0.15 / 1M tokens
    'openai:gpt-4',        # $30 / 1M tokens  ❌ 降级成本反而更高
])
```

**解决方案**：
```python
# 降级链应该按成本递减或质量递减
manager = AgentManager([
    'openai:gpt-4',              # 高质量，贵
    'openai:gpt-4o-mini',        # 中质量，便宜
    'deepseek:deepseek-chat',    # 可用质量，更便宜
])

# 或者设置成本上限
class CostAwareManager(AgentManager):
    def __init__(self, models, max_cost_per_request=0.01):
        super().__init__(models)
        self.max_cost = max_cost_per_request
    
    def estimate_cost(self, model_name, tokens):
        # 根据模型计算成本
        pass
```

---

## 7. 验证你学会了（自测题）

### 练习 1：实现模型池轮询

**需求**：
为了分散请求压力，实现一个 `RoundRobinManager`，每次请求轮流使用不同的模型（而不是降级）。

**提示**：
- 使用 `itertools.cycle` 或手动维护索引
- 保留降级机制：当前模型失败才切换

<details>
<summary>参考答案</summary>

```python
from itertools import cycle
from pydantic_ai import Agent

class RoundRobinManager:
    def __init__(self, model_names):
        self.agents = [(name, Agent(name)) for name in model_names]
        self.cycle = cycle(self.agents)
    
    def run(self, prompt):
        primary_model, primary_agent = next(self.cycle)
        
        try:
            return primary_agent.run_sync(prompt)
        except Exception:
            # 失败了尝试其他模型
            for model_name, agent in self.agents:
                if model_name == primary_model:
                    continue
                try:
                    return agent.run_sync(prompt)
                except Exception:
                    continue
            raise RuntimeError('所有模型都失败')

# 使用
manager = RoundRobinManager(['openai:gpt-4o-mini', 'deepseek:deepseek-chat'])
result1 = manager.run('问题1')  # 用 gpt-4o-mini
result2 = manager.run('问题2')  # 用 deepseek-chat
result3 = manager.run('问题3')  # 又回到 gpt-4o-mini
```

</details>

---

### 练习 2：实现基于成本的智能路由

**需求**：
根据提示词长度自动选择模型：
- 短提示词（< 100 tokens）：用便宜模型
- 长提示词（>= 100 tokens）：用高质量模型

**提示**：
- 使用 `tiktoken` 库计算 token 数
- 在 `AgentManager` 基础上扩展

<details>
<summary>参考答案</summary>

```python
import tiktoken
from pydantic_ai import Agent

class CostOptimizedManager:
    def __init__(self):
        self.cheap_agent = Agent('openai:gpt-4o-mini')
        self.quality_agent = Agent('openai:gpt-4')
        self.encoder = tiktoken.encoding_for_model('gpt-4')
    
    def run(self, prompt):
        tokens = len(self.encoder.encode(prompt))
        
        if tokens < 100:
            print(f'短提示词 ({tokens} tokens)，使用便宜模型')
            return self.cheap_agent.run_sync(prompt)
        else:
            print(f'长提示词 ({tokens} tokens)，使用高质量模型')
            return self.quality_agent.run_sync(prompt)

# 使用
manager = CostOptimizedManager()
manager.run('你好')  # 用 gpt-4o-mini
manager.run('请详细分析...' * 50)  # 用 gpt-4
```

</details>

---

### 练习 3：实现配置热更新

**需求**：
不重启程序的情况下，通过修改 `config.yaml` 文件切换模型。

**提示**：
- 使用 `watchdog` 库监听文件变化
- 或者定期检查文件修改时间

<details>
<summary>参考答案</summary>

```python
import yaml
import time
from pathlib import Path
from pydantic_ai import Agent

class HotReloadManager:
    def __init__(self, config_path='config.yaml'):
        self.config_path = Path(config_path)
        self.last_mtime = 0
        self.agent = None
        self._reload_if_needed()
    
    def _reload_if_needed(self):
        """检查配置文件是否更新"""
        current_mtime = self.config_path.stat().st_mtime
        if current_mtime > self.last_mtime:
            print('配置文件已更新，重新加载...')
            with open(self.config_path, 'r') as f:
                config = yaml.safe_load(f)
            
            model_name = config['models']['primary']
            self.agent = Agent(model_name)
            self.last_mtime = current_mtime
            print(f'当前模型: {model_name}')
    
    def run(self, prompt):
        self._reload_if_needed()
        return self.agent.run_sync(prompt)

# 使用
manager = HotReloadManager()
while True:
    prompt = input('输入问题: ')
    result = manager.run(prompt)
    print(result.data)
```

</details>

---

## 总结

本教程展示了如何使用 Pydantic AI 实现多模型适配和降级策略。核心要点：

1. **统一接口**：`Agent(model_name)` 支持所有主流 LLM
2. **配置驱动**：通过 YAML + 环境变量管理模型
3. **手动降级**：Pydantic AI 没有内置降级，需自己封装
4. **生产实践**：重试、日志、成本控制缺一不可

与原生 API 相比，Pydantic AI 的优势是代码简洁（少写 80% 的适配代码），劣势是需要自己实现降级逻辑（但封装一次后可复用）。

**下一步学习**：
- 阅读 [并发批处理与缓存层-Pydantic-AI版](./并发批处理与缓存层-Pydantic-AI版.md)
- 了解 [长文本处理与输出质量校验-Pydantic-AI版](./长文本处理与输出质量校验-Pydantic-AI版.md)
