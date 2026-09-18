# 并发批处理与缓存层 - Pydantic AI 版

## 1. 它是什么（一句话 + 一句类比）

**一句话定义**：使用 Pydantic AI 的内置并发控制和钩子机制，高效处理大量 LLM 请求并避免重复调用。

**一句类比**：就像快递站的智能分拣系统 — 同时处理 N 个包裹（并发），扫码后发现重复订单直接从仓库取（缓存）。

---

## 2. 它解决什么问题 / 为什么需要它

### 痛点 1：批量处理慢
需要处理 1000 条用户评论的情感分析，如果串行调用 API：
- 每次 2 秒 × 1000 次 = 2000 秒 ≈ 33 分钟
- 用户体验极差，成本也高（长时间占用服务器）

### 痛点 2：重复请求浪费钱
同一个问题被多次调用：
- "总结这篇文章" 可能在 10 分钟内被调 5 次
- 每次都花钱调 API，实际上结果完全一样

### 痛点 3：API 限流
OpenAI 有 RPM（每分钟请求数）和 TPM（每分钟 token 数）限制，并发太高会被拒绝。

### Pydantic AI 如何解决
- **内置并发控制**：`KnownModelName` 自带默认限流，或使用 `ConcurrencyLimitedModel` 手动配置
- **钩子机制**：通过 `@agent.model_call_interceptor` 实现缓存层
- **异步优先**：Pydantic AI 原生支持 `asyncio`，配合 `asyncio.gather` 实现高效并发

---

## 3. 读下去之前，先搞懂这些概念

### 3.1 基础概念

**并发（Concurrency）**  
同时发起多个请求，而不是等一个完成再发下一个。Python 用 `asyncio` 实现。

**批处理（Batch Processing）**  
把大任务拆成小批次，每批控制并发数，避免一次性发太多请求被限流。

**缓存（Cache）**  
保存已计算的结果，相同输入直接返回缓存，避免重复计算。

**TTL（Time To Live）**  
缓存过期时间，超过 TTL 后缓存失效，需要重新计算。

### 3.2 Pydantic AI 核心概念

**`ConcurrencyLimitedModel`**  
限制模型的最大并发请求数：

```python
from pydantic_ai.models import ConcurrencyLimitedModel

# 最多同时发 5 个请求
limited_model = ConcurrencyLimitedModel('openai:gpt-4', max_concurrent=5)
```

**`@agent.model_call_interceptor`**  
在模型调用前后插入自定义逻辑（如缓存、日志、监控）：

```python
from pydantic_ai import Agent

agent = Agent('openai:gpt-4')

@agent.model_call_interceptor
async def cache_interceptor(call, ctx):
    # 调用前：检查缓存
    # 调用后：保存结果
    pass
```

**`run()` 异步调用**  
Pydantic AI 推荐用异步方式处理并发：

```python
import asyncio
from pydantic_ai import Agent

agent = Agent('openai:gpt-4')

async def process_batch(prompts):
    tasks = [agent.run(p) for p in prompts]
    results = await asyncio.gather(*tasks)
    return results
```

---

## 4. 从最简单的代码示例开始

### 示例 1：最小可运行版本（并发处理）

```python
import asyncio
from pydantic_ai import Agent

agent = Agent('openai:gpt-4o-mini')

async def summarize_texts(texts):
    """并发处理多个文本摘要"""
    tasks = [agent.run(f'总结：{text}') for text in texts]
    results = await asyncio.gather(*tasks)
    return [r.data for r in results]

# 使用
texts = ['文本1...', '文本2...', '文本3...']
results = asyncio.run(summarize_texts(texts))
print(results)
```

**效果**：3 个请求同时发出，总耗时约等于单次请求时间（而不是 3 倍）。

**对比原生 API**：
```python
# ❌ 原生 SDK 需要手动管理 asyncio 和 session
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def summarize_texts(texts):
    tasks = [
        client.chat.completions.create(
            model='gpt-4o-mini',
            messages=[{'role': 'user', 'content': f'总结：{text}'}]
        )
        for text in texts
    ]
    results = await asyncio.gather(*tasks)
    return [r.choices[0].message.content for r in results]
```

Pydantic AI 的代码更简洁，不需要处理 `messages` 格式。

---

### 示例 2：限制并发数（避免限流）

```python
import asyncio
from pydantic_ai import Agent
from pydantic_ai.models import ConcurrencyLimitedModel

# 限制最多同时 5 个请求
limited_model = ConcurrencyLimitedModel('openai:gpt-4o-mini', max_concurrent=5)
agent = Agent(limited_model)

async def process_large_batch(texts):
    """处理大批量数据，自动限流"""
    tasks = [agent.run(f'分析情感：{text}') for text in texts]
    results = await asyncio.gather(*tasks)
    return [r.data for r in results]

# 即使有 100 个文本，也只会同时发 5 个请求
texts = [f'用户评论 {i}' for i in range(100)]
results = asyncio.run(process_large_batch(texts))
print(f'处理完成，共 {len(results)} 条结果')
```

**运行效果**：
- 前 5 个请求立即发出
- 第 6 个等第 1 个完成后才发出
- 以此类推，保持最多 5 个并发

---

### 示例 3：加上缓存层

```python
import asyncio
import hashlib
import time
from typing import Dict, Tuple
from pydantic_ai import Agent

class CachedAgent:
    """带 TTL 缓存的 Agent"""
    
    def __init__(self, model_name: str, cache_ttl: int = 3600):
        self.agent = Agent(model_name)
        self.cache: Dict[str, Tuple[str, float]] = {}  # {cache_key: (result, timestamp)}
        self.cache_ttl = cache_ttl
        self.hit_count = 0
        self.miss_count = 0
    
    def _generate_cache_key(self, prompt: str) -> str:
        """生成缓存键"""
        return hashlib.md5(prompt.encode()).hexdigest()
    
    def _is_cache_valid(self, timestamp: float) -> bool:
        """检查缓存是否过期"""
        return (time.time() - timestamp) < self.cache_ttl
    
    async def run(self, prompt: str) -> str:
        """带缓存的运行"""
        cache_key = self._generate_cache_key(prompt)
        
        # 检查缓存
        if cache_key in self.cache:
            result, timestamp = self.cache[cache_key]
            if self._is_cache_valid(timestamp):
                self.hit_count += 1
                print(f'✅ 缓存命中: {prompt[:30]}...')
                return result
            else:
                print(f'⏰ 缓存过期: {prompt[:30]}...')
                del self.cache[cache_key]
        
        # 缓存未命中，调用 API
        self.miss_count += 1
        print(f'❌ 缓存未命中，调用 API: {prompt[:30]}...')
        result = await self.agent.run(prompt)
        
        # 保存到缓存
        self.cache[cache_key] = (result.data, time.time())
        return result.data
    
    def get_stats(self) -> dict:
        """获取缓存统计"""
        total = self.hit_count + self.miss_count
        hit_rate = (self.hit_count / total * 100) if total > 0 else 0
        return {
            'hit': self.hit_count,
            'miss': self.miss_count,
            'hit_rate': f'{hit_rate:.1f}%',
            'cache_size': len(self.cache)
        }


# 使用示例
async def main():
    agent = CachedAgent('openai:gpt-4o-mini', cache_ttl=60)  # 缓存 60 秒
    
    # 第一次调用
    result1 = await agent.run('什么是 Python')
    print(f'结果1: {result1[:50]}...\n')
    
    # 第二次调用相同问题（命中缓存）
    result2 = await agent.run('什么是 Python')
    print(f'结果2: {result2[:50]}...\n')
    
    # 不同问题（缓存未命中）
    result3 = await agent.run('什么是 JavaScript')
    print(f'结果3: {result3[:50]}...\n')
    
    # 统计
    print('缓存统计:', agent.get_stats())

asyncio.run(main())
```

**运行输出**：
```
❌ 缓存未命中，调用 API: 什么是 Python...
结果1: Python 是一种高级编程语言...

✅ 缓存命中: 什么是 Python...
结果2: Python 是一种高级编程语言...

❌ 缓存未命中，调用 API: 什么是 JavaScript...
结果3: JavaScript 是一种脚本语言...

缓存统计: {'hit': 1, 'miss': 2, 'hit_rate': '33.3%', 'cache_size': 2}
```

---

## 5. 最小可用的实际用法（生产场景模板）

### 5.1 完整项目结构

```
batch_processor/
├── .env
├── config.yaml
├── requirements.txt
├── src/
│   ├── __init__.py
│   ├── cached_agent.py       # 带缓存的 Agent
│   ├── batch_processor.py    # 批处理引擎
│   └── progress_tracker.py   # 进度跟踪
├── cli.py
└── tests/
    └── test_cache.py
```

### 5.2 核心代码 `src/cached_agent.py`

```python
import asyncio
import hashlib
import time
import logging
from typing import Dict, Tuple, Optional
from pydantic_ai import Agent
from pydantic_ai.models import ConcurrencyLimitedModel

logger = logging.getLogger(__name__)


class ProductionCachedAgent:
    """生产级带缓存和并发控制的 Agent"""
    
    def __init__(
        self,
        model_name: str,
        max_concurrent: int = 5,
        cache_ttl: int = 3600,
        cache_max_size: int = 1000
    ):
        """
        Args:
            model_name: 模型标识符
            max_concurrent: 最大并发数
            cache_ttl: 缓存过期时间（秒）
            cache_max_size: 缓存最大条目数
        """
        # 创建限流模型
        limited_model = ConcurrencyLimitedModel(model_name, max_concurrent=max_concurrent)
        self.agent = Agent(limited_model)
        
        # 缓存配置
        self.cache: Dict[str, Tuple[str, float]] = {}
        self.cache_ttl = cache_ttl
        self.cache_max_size = cache_max_size
        
        # 统计信息
        self.stats = {
            'hit': 0,
            'miss': 0,
            'api_calls': 0,
            'errors': 0
        }
    
    def _generate_cache_key(self, prompt: str, system_prompt: Optional[str] = None) -> str:
        """生成缓存键（考虑 system_prompt）"""
        content = prompt + (system_prompt or '')
        return hashlib.sha256(content.encode()).hexdigest()
    
    def _is_cache_valid(self, timestamp: float) -> bool:
        """检查缓存是否过期"""
        return (time.time() - timestamp) < self.cache_ttl
    
    def _evict_oldest(self):
        """缓存满了，移除最旧的条目"""
        if len(self.cache) >= self.cache_max_size:
            oldest_key = min(self.cache.keys(), key=lambda k: self.cache[k][1])
            del self.cache[oldest_key]
            logger.debug(f'缓存已满，移除最旧条目: {oldest_key[:8]}')
    
    async def run(self, prompt: str, system_prompt: Optional[str] = None) -> str:
        """
        带缓存的运行
        
        Args:
            prompt: 用户提示词
            system_prompt: 系统提示词（可选）
        
        Returns:
            模型输出结果
        """
        cache_key = self._generate_cache_key(prompt, system_prompt)
        
        # 检查缓存
        if cache_key in self.cache:
            result, timestamp = self.cache[cache_key]
            if self._is_cache_valid(timestamp):
                self.stats['hit'] += 1
                logger.info(f'缓存命中: {cache_key[:8]}')
                return result
            else:
                # 过期删除
                del self.cache[cache_key]
        
        # 缓存未命中，调用 API
        self.stats['miss'] += 1
        self.stats['api_calls'] += 1
        logger.info(f'调用 API: {prompt[:50]}...')
        
        try:
            if system_prompt:
                # 需要创建新 Agent（Pydantic AI 不支持动态修改 system_prompt）
                temp_agent = Agent(
                    self.agent._model,
                    system_prompt=system_prompt
                )
                result = await temp_agent.run(prompt)
            else:
                result = await self.agent.run(prompt)
            
            # 保存到缓存
            self._evict_oldest()
            self.cache[cache_key] = (result.data, time.time())
            
            return result.data
        
        except Exception as e:
            self.stats['errors'] += 1
            logger.error(f'API 调用失败: {e}')
            raise
    
    def get_stats(self) -> dict:
        """获取统计信息"""
        total = self.stats['hit'] + self.stats['miss']
        hit_rate = (self.stats['hit'] / total * 100) if total > 0 else 0
        
        return {
            **self.stats,
            'hit_rate': f'{hit_rate:.1f}%',
            'cache_size': len(self.cache),
            'cache_limit': self.cache_max_size
        }
    
    def clear_cache(self):
        """清空缓存"""
        self.cache.clear()
        logger.info('缓存已清空')
```

### 5.3 批处理引擎 `src/batch_processor.py`

```python
import asyncio
import logging
from typing import List, Dict, Any, Callable
from dataclasses import dataclass
from .cached_agent import ProductionCachedAgent

logger = logging.getLogger(__name__)


@dataclass
class BatchResult:
    """批处理结果"""
    success: List[Dict[str, Any]]
    failed: List[Dict[str, Any]]
    total: int
    success_rate: float


class BatchProcessor:
    """批量处理引擎"""
    
    def __init__(
        self,
        agent: ProductionCachedAgent,
        batch_size: int = 50,
        retry_failed: bool = True,
        max_retries: int = 2
    ):
        """
        Args:
            agent: CachedAgent 实例
            batch_size: 每批处理数量
            retry_failed: 是否重试失败项
            max_retries: 最大重试次数
        """
        self.agent = agent
        self.batch_size = batch_size
        self.retry_failed = retry_failed
        self.max_retries = max_retries
    
    async def process_batch(
        self,
        items: List[str],
        transform_fn: Callable[[str], str] = None,
        progress_callback: Callable[[int, int], None] = None
    ) -> BatchResult:
        """
        批量处理
        
        Args:
            items: 待处理的项目列表
            transform_fn: 转换函数，将项目转为提示词
            progress_callback: 进度回调 (completed, total)
        
        Returns:
            BatchResult 包含成功和失败的结果
        """
        if transform_fn is None:
            transform_fn = lambda x: x  # 默认直接使用
        
        total = len(items)
        success_results = []
        failed_results = []
        
        # 分批处理
        for i in range(0, total, self.batch_size):
            batch = items[i:i + self.batch_size]
            logger.info(f'处理批次 {i // self.batch_size + 1}: {len(batch)} 项')
            
            # 并发处理当前批次
            tasks = []
            for idx, item in enumerate(batch):
                prompt = transform_fn(item)
                tasks.append(self._process_single(item, prompt, i + idx))
            
            batch_results = await asyncio.gather(*tasks, return_exceptions=True)
            
            # 分类结果
            for result in batch_results:
                if isinstance(result, Exception):
                    failed_results.append({
                        'item': None,
                        'error': str(result)
                    })
                elif result['success']:
                    success_results.append(result)
                else:
                    failed_results.append(result)
            
            # 进度回调
            if progress_callback:
                progress_callback(i + len(batch), total)
        
        # 重试失败项
        if self.retry_failed and failed_results:
            logger.info(f'重试 {len(failed_results)} 个失败项')
            failed_results = await self._retry_failed(failed_results, transform_fn)
        
        success_rate = len(success_results) / total * 100 if total > 0 else 0
        
        return BatchResult(
            success=success_results,
            failed=failed_results,
            total=total,
            success_rate=success_rate
        )
    
    async def _process_single(self, item: Any, prompt: str, index: int) -> Dict[str, Any]:
        """处理单个项目"""
        try:
            result = await self.agent.run(prompt)
            return {
                'success': True,
                'index': index,
                'item': item,
                'result': result
            }
        except Exception as e:
            logger.warning(f'项目 {index} 失败: {e}')
            return {
                'success': False,
                'index': index,
                'item': item,
                'error': str(e)
            }
    
    async def _retry_failed(
        self,
        failed_items: List[Dict],
        transform_fn: Callable
    ) -> List[Dict]:
        """重试失败的项目"""
        remaining_failed = []
        
        for attempt in range(self.max_retries):
            if not failed_items:
                break
            
            logger.info(f'重试尝试 {attempt + 1}/{self.max_retries}')
            await asyncio.sleep(2 ** attempt)  # 指数退避
            
            retry_tasks = []
            for failed in failed_items:
                if 'item' in failed and failed['item']:
                    prompt = transform_fn(failed['item'])
                    retry_tasks.append(
                        self._process_single(
                            failed['item'],
                            prompt,
                            failed.get('index', -1)
                        )
                    )
            
            if retry_tasks:
                retry_results = await asyncio.gather(*retry_tasks, return_exceptions=True)
                
                # 更新失败列表
                failed_items = []
                for result in retry_results:
                    if isinstance(result, Exception) or not result.get('success'):
                        failed_items.append(result if isinstance(result, dict) else {'error': str(result)})
        
        return failed_items
```

### 5.4 CLI 入口 `cli.py`

```python
import asyncio
import click
import logging
from pathlib import Path
from src.cached_agent import ProductionCachedAgent
from src.batch_processor import BatchProcessor

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)


@click.command()
@click.argument('input_file', type=click.Path(exists=True))
@click.option('--output', '-o', default='output.txt', help='输出文件路径')
@click.option('--model', '-m', default='openai:gpt-4o-mini', help='模型名称')
@click.option('--concurrent', '-c', default=5, help='最大并发数')
@click.option('--batch-size', '-b', default=50, help='批次大小')
@click.option('--cache-ttl', default=3600, help='缓存过期时间（秒）')
def main(input_file, output, model, concurrent, batch_size, cache_ttl):
    """批量处理文本文件"""
    
    # 读取输入
    with open(input_file, 'r', encoding='utf-8') as f:
        items = [line.strip() for line in f if line.strip()]
    
    logger.info(f'读取 {len(items)} 条数据')
    
    # 初始化
    agent = ProductionCachedAgent(
        model_name=model,
        max_concurrent=concurrent,
        cache_ttl=cache_ttl
    )
    
    processor = BatchProcessor(agent, batch_size=batch_size)
    
    # 进度回调
    def on_progress(completed, total):
        percent = completed / total * 100
        click.echo(f'进度: {completed}/{total} ({percent:.1f}%)')
    
    # 执行
    result = asyncio.run(processor.process_batch(
        items,
        transform_fn=lambda x: f'总结以下文本：{x}',
        progress_callback=on_progress
    ))
    
    # 保存结果
    with open(output, 'w', encoding='utf-8') as f:
        for item in result.success:
            f.write(f"{item['result']}\n")
    
    # 统计
    stats = agent.get_stats()
    click.echo(f"\n✅ 完成!")
    click.echo(f"成功: {len(result.success)}/{result.total} ({result.success_rate:.1f}%)")
    click.echo(f"失败: {len(result.failed)}")
    click.echo(f"缓存命中率: {stats['hit_rate']}")
    click.echo(f"API 调用次数: {stats['api_calls']}")


if __name__ == '__main__':
    main()
```

**使用示例**：
```bash
# 处理 1000 条用户评论
python cli.py reviews.txt -o results.txt --concurrent 10 --batch-size 100

# 使用不同模型
python cli.py texts.txt --model deepseek:deepseek-chat --concurrent 20
```

### 5.5 依赖文件

```txt
pydantic-ai==0.0.14
pydantic==2.10.4
openai==1.57.4
click==8.1.8
python-dotenv==1.0.1
```

---

## 6. 常见坑与易错点

### 坑点 1：`ConcurrencyLimitedModel` 只控制并发数，不支持 RPM 限流

**问题**：
```python
# ❌ 这样只能控制同时发 5 个请求，但如果每个请求 0.5 秒
# 1 分钟内会发 5 * 120 = 600 个请求，超过 OpenAI 的 RPM 限制
limited_model = ConcurrencyLimitedModel('openai:gpt-4', max_concurrent=5)
```

**解决方案**：结合 `asyncio.Semaphore` 实现窗口限流
```python
import asyncio
import time

class RPMLimitedAgent:
    def __init__(self, agent, rpm_limit=60):
        self.agent = agent
        self.rpm_limit = rpm_limit
        self.request_times = []
    
    async def run(self, prompt):
        # 清理 1 分钟前的记录
        current_time = time.time()
        self.request_times = [t for t in self.request_times if current_time - t < 60]
        
        # 检查是否超限
        if len(self.request_times) >= self.rpm_limit:
            wait_time = 60 - (current_time - self.request_times[0])
            print(f'RPM 限制，等待 {wait_time:.1f} 秒')
            await asyncio.sleep(wait_time)
        
        # 记录请求时间
        self.request_times.append(time.time())
        return await self.agent.run(prompt)
```

---

### 坑点 2：缓存 key 生成需考虑模型参数

**问题**：
```python
# ❌ 只用 prompt 作为 key，忽略了 temperature 等参数
cache_key = hashlib.md5(prompt.encode()).hexdigest()

# 结果：temperature=0.0 和 temperature=1.0 返回相同缓存（错误）
```

**解决方案**：
```python
def _generate_cache_key(self, prompt, temperature=0.7, max_tokens=1000):
    # 包含所有影响输出的参数
    content = f'{prompt}|temp:{temperature}|max:{max_tokens}'
    return hashlib.sha256(content.encode()).hexdigest()
```

---

### 坑点 3：异步场景下的缓存竞态条件

**问题**：
```python
# ❌ 两个并发请求同时发现缓存未命中，都去调 API
# 结果：浪费了一次 API 调用
async def run(self, prompt):
    if prompt not in self.cache:  # 检查和调用之间有时间窗口
        result = await self.agent.run(prompt)
        self.cache[prompt] = result
```

**解决方案**：使用 `asyncio.Lock`
```python
import asyncio

class ThreadSafeCachedAgent:
    def __init__(self, agent):
        self.agent = agent
        self.cache = {}
        self.locks = {}  # 每个 prompt 一个锁
    
    async def run(self, prompt):
        if prompt not in self.locks:
            self.locks[prompt] = asyncio.Lock()
        
        async with self.locks[prompt]:
            if prompt in self.cache:
                return self.cache[prompt]
            
            result = await self.agent.run(prompt)
            self.cache[prompt] = result.data
            return result.data
```

---

### 坑点 4：内存泄漏 - 缓存无限增长

**问题**：
```python
# ❌ 长时间运行后，缓存占用几个 GB 内存
self.cache = {}  # 没有大小限制
```

**解决方案**：
- 方案 1：限制缓存条目数（示例 5.2 已实现）
- 方案 2：使用 LRU 缓存
```python
from functools import lru_cache

class LRUCachedAgent:
    def __init__(self, agent, maxsize=1000):
        self.agent = agent
        self._run_cached = lru_cache(maxsize=maxsize)(self._run_sync_wrapper)
    
    def _run_sync_wrapper(self, prompt):
        # lru_cache 不支持异步，需要同步包装
        return asyncio.run(self.agent.run(prompt))
```

---

### 坑点 5：进度跟踪不准确

**问题**：
```python
# ❌ asyncio.gather 无法获取实时进度
results = await asyncio.gather(*tasks)
# 用户只能看到 "处理中..." 然后突然完成
```

**解决方案**：使用 `asyncio.as_completed`
```python
async def process_with_progress(self, items):
    tasks = {asyncio.create_task(self.agent.run(item)): item for item in items}
    
    completed = 0
    for coro in asyncio.as_completed(tasks.keys()):
        result = await coro
        completed += 1
        print(f'进度: {completed}/{len(items)} ({completed/len(items)*100:.1f}%)')
        yield result
```

---

## 7. 验证你学会了（自测题）

### 练习 1：实现批次间延迟

**需求**：
为了更精细地控制 RPM，在每个批次处理完成后暂停 N 秒再处理下一批。

**提示**：
在 `BatchProcessor.process_batch` 的批次循环中加入 `asyncio.sleep`

<details>
<summary>参考答案</summary>

```python
class BatchProcessor:
    def __init__(self, agent, batch_size=50, batch_delay=5):
        self.agent = agent
        self.batch_size = batch_size
        self.batch_delay = batch_delay
    
    async def process_batch(self, items):
        total = len(items)
        results = []
        
        for i in range(0, total, self.batch_size):
            batch = items[i:i + self.batch_size]
            tasks = [self.agent.run(item) for item in batch]
            batch_results = await asyncio.gather(*tasks)
            results.extend(batch_results)
            
            # 批次间延迟
            if i + self.batch_size < total:
                print(f'批次完成，暂停 {self.batch_delay} 秒...')
                await asyncio.sleep(self.batch_delay)
        
        return results
```

</details>

---

### 练习 2：实现缓存预热

**需求**：
在程序启动时，从文件加载常见问题的缓存，避免冷启动时的高延迟。

**提示**：
缓存文件格式：`prompt|result|timestamp`

<details>
<summary>参考答案</summary>

```python
import json
import time

class CachedAgent:
    def __init__(self, agent, cache_file='cache.json'):
        self.agent = agent
        self.cache = {}
        self.cache_file = cache_file
        self.load_cache()
    
    def load_cache(self):
        """从文件加载缓存"""
        try:
            with open(self.cache_file, 'r', encoding='utf-8') as f:
                data = json.load(f)
                for item in data:
                    self.cache[item['key']] = (item['result'], item['timestamp'])
                print(f'加载 {len(data)} 条缓存')
        except FileNotFoundError:
            print('缓存文件不存在，跳过预热')
    
    def save_cache(self):
        """保存缓存到文件"""
        data = [
            {'key': k, 'result': v[0], 'timestamp': v[1]}
            for k, v in self.cache.items()
        ]
        with open(self.cache_file, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        print(f'保存 {len(data)} 条缓存')
```

</details>

---

### 练习 3：实现失败重试的指数退避

**需求**：
批处理时，如果某个请求失败，按 2^n 秒的间隔重试（1秒、2秒、4秒...）。

**提示**：
在 `_process_single` 中加入重试逻辑

<details>
<summary>参考答案</summary>

```python
async def _process_single_with_retry(self, item, prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = await self.agent.run(prompt)
            return {'success': True, 'item': item, 'result': result}
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f'失败，{wait_time} 秒后重试')
                await asyncio.sleep(wait_time)
            else:
                return {'success': False, 'item': item, 'error': str(e)}
```

</details>

---

## 总结

本教程展示了如何使用 Pydantic AI 实现高效的并发批处理和缓存层。核心要点：

1. **并发控制**：`ConcurrencyLimitedModel` + `asyncio.gather`
2. **缓存策略**：自己实现 TTL 缓存，注意线程安全
3. **批处理**：分批 + 进度跟踪 + 失败重试
4. **生产实践**：内存管理、RPM 限流、错误处理

与原生 API 相比，Pydantic AI 的优势是并发控制开箱即用，劣势是缓存和高级限流需要自己实现（但封装一次后可复用）。

**性能对比**：
- 原生串行：1000 条 × 2 秒 = 2000 秒
- Pydantic AI 并发（10 并发）：1000 ÷ 10 × 2 秒 = 200 秒（快 10 倍）
- 加缓存（50% 命中率）：200 × 0.5 = 100 秒（再快 1 倍）

**下一步学习**：
- 阅读 [长文本处理与输出质量校验-Pydantic-AI版](./长文本处理与输出质量校验-Pydantic-AI版.md)
- 了解 [AI应用安全与交付形态-Pydantic-AI版](./AI应用安全与交付形态-Pydantic-AI版.md)
