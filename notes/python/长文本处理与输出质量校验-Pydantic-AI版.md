# 长文本处理与输出质量校验 - Pydantic AI 版

## 1. 它是什么（一句话 + 一句类比）

**一句话定义**：把超出模型上下文窗口的长文本切成块分别处理再汇总（map-reduce），同时用 Pydantic AI 的 `output_type` + `@agent.output_validator` 让不合格的输出自动重试。

**一句类比**：像流水线质检 — 大批货物分箱处理（分块），每箱出来先过质检机（校验器），不合格的直接退回重做（自动重试），最后合并装车（reduce）。

---

## 2. 它解决什么问题 / 为什么需要它

### 痛点 1：文本太长塞不进去
一份 80 页的 PDF 合同大约 60k tokens，超过很多模型的实际可用窗口；即便塞得进去，长上下文的准确率也会明显下降，中间部分容易被"忽略"。

### 痛点 2：输出格式不稳定
你要 JSON，模型给你带 markdown 代码块的 JSON；你要 3 个要点，模型给 1 个。下游代码一 `json.loads` 就炸。

### 痛点 3：手写重试循环又长又脏
原生 API 的做法通常是：调用 → 解析 → 校验 → 失败就拼一段"你上次格式错了"的提示词 → 再调用。这段逻辑每个项目都要重写一遍。

### Pydantic AI 如何解决
- **结构化输出**：`output_type=YourModel`，框架负责把模型输出解析成 Pydantic 对象，解析失败自动把错误回传给模型重试
- **业务校验**：`@agent.output_validator` 里 `raise ModelRetry('原因')`，框架自动带着原因重试，不用自己拼提示词
- **重试上限**：`retries=N` 一个参数搞定
- **分块仍需手动**：这部分 Pydantic AI 不管，用 `langchain-text-splitters` 单独拿来用

---

## 3. 读下去之前，先搞懂这些概念

### 3.1 基础概念

**上下文窗口（Context Window）**  
模型单次能处理的 token 上限，输入 + 输出共享这个额度。

**分块（Chunking）**  
把长文本按 token 数切成若干块。相邻块留一点**重叠（overlap）**，避免把一句话劈成两半导致语义断裂。

**Map-Reduce**  
- Map：对每个块独立处理（例如各自摘要）
- Reduce：把所有块的结果合并成最终答案

**结构化输出（Structured Output）**  
让模型返回符合特定 schema 的数据（而不是自由文本），下游代码可以直接用。

### 3.2 Pydantic AI 核心概念

**`output_type`**  
声明期望的输出类型，Pydantic AI 会自动把模型输出解析成该类型：

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class Summary(BaseModel):
    title: str
    key_points: list[str]

agent = Agent('openai:gpt-4o', output_type=Summary)
result = agent.run_sync('总结这篇文章：...')
print(result.output.title)        # 直接拿字段，不用 json.loads
print(result.output.key_points)   # list[str]
```

> 注意：结果通过 `result.output` 访问。旧版本（0.0.x）里叫 `result.data`，新版本已改名为 `output`，`data` 是废弃别名。

**`retries`**  
最大重试次数（包含解析失败和校验失败）：

```python
agent = Agent('openai:gpt-4o', output_type=Summary, retries=3)
```

**`@agent.output_validator`**  
业务级校验钩子。校验不通过时 `raise ModelRetry(...)`，框架会把这条消息作为反馈发回模型让它重做：

```python
from pydantic_ai import ModelRetry

@agent.output_validator
def check_points(output: Summary) -> Summary:
    if len(output.key_points) < 3:
        raise ModelRetry('要点少于 3 条，请补充到至少 3 条')
    return output
```

**`ModelRetry` vs 普通异常**  
- `ModelRetry`：告诉框架"这次输出不行，带着原因重试"
- 其他异常：直接向上抛出，不重试

**`RunContext`**  
运行上下文，校验器可以声明它来拿到本次运行的信息（用了几次重试、当前模型等）：

```python
from pydantic_ai import RunContext

@agent.output_validator
def check(ctx: RunContext[None], output: Summary) -> Summary:
    print(f'当前是第 {ctx.retry} 次重试')
    return output
```

**`UnexpectedModelBehavior`**  
重试次数用尽后 Pydantic AI 抛出的异常，需要在业务侧兜底。

---

## 4. 从最简单的代码示例开始

### 示例 1：最小可运行版本（结构化输出）

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent


class Summary(BaseModel):
    title: str = Field(description='一句话标题，不超过 20 字')
    key_points: list[str] = Field(description='核心要点列表')


agent = Agent('openai:gpt-4o-mini', output_type=Summary)

article = '''
Python 的异步编程基于事件循环。asyncio 提供了 async/await 语法，
让 I/O 密集型任务可以在等待期间让出控制权，从而提升吞吐量。
但要注意，异步不会让 CPU 密集型任务变快。
'''

result = agent.run_sync(f'总结以下文章：\n{article}')

print(result.output.title)
for i, point in enumerate(result.output.key_points, 1):
    print(f'{i}. {point}')
```

**输出**：
```
Python 异步编程要点
1. asyncio 基于事件循环实现异步
2. async/await 让 I/O 等待期间可以让出控制权
3. 异步无法加速 CPU 密集型任务
```

**关键点**：`result.output` 已经是 `Summary` 实例，不需要 `json.loads`，也不需要处理模型偶尔多包一层 ```json 代码块的情况 — 框架已经处理了。

**对比原生 API**：
```python
# ❌ 原生 SDK 的等价实现
import json
from openai import OpenAI

client = OpenAI()
resp = client.chat.completions.create(
    model='gpt-4o-mini',
    messages=[{'role': 'user', 'content': f'总结以下文章，返回 JSON...\n{article}'}],
    response_format={'type': 'json_object'},
)
raw = resp.choices[0].message.content
data = json.loads(raw)          # 可能抛 JSONDecodeError
summary = Summary(**data)       # 可能抛 ValidationError
# 上面两处异常都要自己接住、自己拼重试提示词、自己数重试次数
```

---

### 示例 2：加一个业务校验（自动重试）

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent, ModelRetry, RunContext


class Summary(BaseModel):
    title: str = Field(description='一句话标题')
    key_points: list[str] = Field(description='核心要点')


agent = Agent(
    'openai:gpt-4o-mini',
    output_type=Summary,
    retries=3,  # 最多重试 3 次
)


@agent.output_validator
def validate_summary(ctx: RunContext[None], output: Summary) -> Summary:
    """业务规则：至少 3 个要点，标题不超过 20 字"""
    if len(output.key_points) < 3:
        raise ModelRetry(
            f'只给了 {len(output.key_points)} 个要点，需要至少 3 个，请补充'
        )

    if len(output.title) > 20:
        raise ModelRetry(f'标题 {len(output.title)} 字太长了，压缩到 20 字以内')

    # 去掉空白要点
    output.key_points = [p.strip() for p in output.key_points if p.strip()]
    return output


result = agent.run_sync(f'总结：{article}')
print(result.output)
```

**重试是怎么发生的**：  
校验器 `raise ModelRetry('只给了 1 个要点...')` 之后，Pydantic AI 会把这条消息作为一条新的用户消息追加到对话里，再次请求模型。模型看到具体原因，第二次通常就能改对。整个过程你不需要写循环。

**查看重试过程**：
```python
result = agent.run_sync(f'总结：{article}')
for msg in result.all_messages():
    print(msg)   # 能看到失败的那次输出和回传的重试提示
```

---

### 示例 3：贴近项目的样子（长文本 map-reduce）

```python
import asyncio
from pydantic import BaseModel, Field
from pydantic_ai import Agent, ModelRetry
from langchain_text_splitters import RecursiveCharacterTextSplitter


class ChunkSummary(BaseModel):
    """单个分块的摘要"""
    points: list[str] = Field(description='这一段的要点，2-4 条')


class FinalSummary(BaseModel):
    """最终汇总结果"""
    title: str = Field(description='整篇文档的标题')
    overview: str = Field(description='一段总览，100 字左右')
    key_points: list[str] = Field(description='去重后的核心要点，5-8 条')


# Map 阶段的 Agent：处理单个分块
map_agent = Agent(
    'openai:gpt-4o-mini',   # 分块多，用便宜模型
    output_type=ChunkSummary,
    retries=2,
    system_prompt='你是文档摘要助手。只提炼给定片段里明确写到的信息，不要推测。',
)


@map_agent.output_validator
def validate_chunk(output: ChunkSummary) -> ChunkSummary:
    if not output.points:
        raise ModelRetry('要点为空，请至少提炼 2 条')
    return output


# Reduce 阶段的 Agent：合并所有分块结果
reduce_agent = Agent(
    'openai:gpt-4o',        # 汇总质量要求高，用好模型
    output_type=FinalSummary,
    retries=3,
    system_prompt='你负责把多段摘要合并成一份完整摘要，合并同类项、去掉重复。',
)


@reduce_agent.output_validator
def validate_final(output: FinalSummary) -> FinalSummary:
    if len(output.key_points) < 5:
        raise ModelRetry(f'核心要点只有 {len(output.key_points)} 条，需要 5-8 条')
    if len(output.overview) < 50:
        raise ModelRetry('总览太短，写到 100 字左右')
    return output


def split_text(text: str, chunk_size: int = 3000, overlap: int = 200) -> list[str]:
    """按字符数分块，相邻块保留重叠避免语义断裂"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        separators=['\n\n', '\n', '。', '！', '？', ' ', ''],  # 中文友好
    )
    return splitter.split_text(text)


async def summarize_long_text(text: str, max_concurrent: int = 5) -> FinalSummary:
    """长文本 map-reduce 摘要"""
    chunks = split_text(text)
    print(f'分成 {len(chunks)} 块')

    # Map：并发处理所有分块，用信号量控制并发
    sem = asyncio.Semaphore(max_concurrent)

    async def run_one(idx: int, chunk: str) -> list[str]:
        async with sem:
            try:
                result = await map_agent.run(f'提炼以下片段的要点：\n\n{chunk}')
                print(f'  块 {idx + 1}/{len(chunks)} 完成')
                return result.output.points
            except Exception as e:
                print(f'  块 {idx + 1} 失败，跳过：{e}')
                return []

    chunk_results = await asyncio.gather(
        *[run_one(i, c) for i, c in enumerate(chunks)]
    )

    # 摊平所有要点
    all_points = [p for points in chunk_results for p in points]
    if not all_points:
        raise RuntimeError('所有分块都处理失败了')

    print(f'Map 阶段共得到 {len(all_points)} 条要点，开始 Reduce')

    # Reduce：合并
    joined = '\n'.join(f'- {p}' for p in all_points)
    final = await reduce_agent.run(f'把以下要点合并成一份完整摘要：\n\n{joined}')
    return final.output


if __name__ == '__main__':
    long_text = open('report.txt', encoding='utf-8').read()
    summary = asyncio.run(summarize_long_text(long_text))

    print(f'\n标题：{summary.title}')
    print(f'\n总览：{summary.overview}')
    print('\n核心要点：')
    for i, p in enumerate(summary.key_points, 1):
        print(f'{i}. {p}')
```

**设计要点**：
- Map 用便宜模型（分块数量多），Reduce 用好模型（只调一次，质量优先）
- 单块失败返回空列表而不是抛异常，避免一块坏掉整个任务失败
- `asyncio.Semaphore` 控并发，避免一次性打爆 API 限流
- 中文文本要在 `separators` 里加 `。！？`，否则默认分隔符对中文效果差

---

## 5. 最小可用的实际用法（生产场景模板）

### 5.1 完整项目结构

```
doc_summarizer/
├── .env
├── requirements.txt
├── src/
│   ├── __init__.py
│   ├── schemas.py        # Pydantic 输出模型
│   ├── splitter.py       # 分块逻辑
│   └── summarizer.py     # map-reduce 引擎
├── cli.py
└── tests/
    └── test_summarizer.py
```

### 5.2 输出模型 `src/schemas.py`

```python
from pydantic import BaseModel, Field


class ChunkSummary(BaseModel):
    """单个分块的摘要结果"""
    points: list[str] = Field(description='这一段的要点，2-4 条，每条不超过 50 字')


class FinalSummary(BaseModel):
    """最终汇总结果"""
    title: str = Field(description='整篇文档标题，不超过 30 字')
    overview: str = Field(description='总览段落，100-150 字')
    key_points: list[str] = Field(description='去重合并后的核心要点，5-8 条')
    keywords: list[str] = Field(description='关键词，3-6 个')


class SummarizeReport(BaseModel):
    """带元信息的处理报告，供上层调用方使用"""
    summary: FinalSummary
    chunk_total: int
    chunk_failed: int
    map_retries: int
```

### 5.3 分块逻辑 `src/splitter.py`

```python
import tiktoken
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 中文友好的分隔符优先级：段落 > 换行 > 句号 > 空格 > 字符
CN_SEPARATORS = ['\n\n', '\n', '。', '！', '？', '；', ' ', '']


def split_by_chars(text: str, chunk_size: int = 3000, overlap: int = 200) -> list[str]:
    """按字符数分块（简单、快，适合大多数场景）"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        separators=CN_SEPARATORS,
    )
    return splitter.split_text(text)


def split_by_tokens(
    text: str,
    chunk_tokens: int = 1500,
    overlap_tokens: int = 100,
    encoding_name: str = 'cl100k_base',
) -> list[str]:
    """按 token 数分块（精确，适合需要严格控制成本的场景）"""
    splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
        encoding_name=encoding_name,
        chunk_size=chunk_tokens,
        chunk_overlap=overlap_tokens,
        separators=CN_SEPARATORS,
    )
    return splitter.split_text(text)


def count_tokens(text: str, encoding_name: str = 'cl100k_base') -> int:
    """估算 token 数，用于成本预估"""
    encoder = tiktoken.get_encoding(encoding_name)
    return len(encoder.encode(text))
```

**两种分块方式怎么选**：
- 按字符：中文 1 字 ≈ 1.5-2 token，估算够用，速度快
- 按 token：需要严格控制单次请求成本，或文本中英混杂时用这个

### 5.4 map-reduce 引擎 `src/summarizer.py`

```python
import asyncio
import logging
from pydantic_ai import Agent, ModelRetry, RunContext
from pydantic_ai.exceptions import UnexpectedModelBehavior

from .schemas import ChunkSummary, FinalSummary, SummarizeReport
from .splitter import split_by_tokens, count_tokens

logger = logging.getLogger(__name__)


class LongTextSummarizer:
    """长文本摘要引擎：分块 → 并发 map → reduce 汇总"""

    def __init__(
        self,
        map_model: str = 'openai:gpt-4o-mini',
        reduce_model: str = 'openai:gpt-4o',
        chunk_tokens: int = 1500,
        max_concurrent: int = 5,
        map_retries: int = 2,
        reduce_retries: int = 3,
    ):
        self.chunk_tokens = chunk_tokens
        self.max_concurrent = max_concurrent
        self._retry_count = 0

        self.map_agent = Agent(
            map_model,
            output_type=ChunkSummary,
            retries=map_retries,
            system_prompt=(
                '你是文档摘要助手。只提炼片段中明确写到的信息，不要推测或补充外部知识。'
                '每条要点独立成句，不超过 50 字。'
            ),
        )

        self.reduce_agent = Agent(
            reduce_model,
            output_type=FinalSummary,
            retries=reduce_retries,
            system_prompt=(
                '你负责把多段摘要合并成一份完整摘要。合并同类项、删除重复、'
                '保持原文的事实，不要引入新信息。'
            ),
        )

        self._register_validators()

    def _register_validators(self):
        """注册输出校验器"""

        @self.map_agent.output_validator
        def validate_chunk(ctx: RunContext[None], output: ChunkSummary) -> ChunkSummary:
            self._retry_count = max(self._retry_count, ctx.retry)

            points = [p.strip() for p in output.points if p.strip()]
            if len(points) < 2:
                raise ModelRetry(f'有效要点只有 {len(points)} 条，请提炼 2-4 条')

            too_long = [p for p in points if len(p) > 80]
            if too_long:
                raise ModelRetry(f'有 {len(too_long)} 条要点超过 80 字，请压缩到 50 字左右')

            output.points = points
            return output

        @self.reduce_agent.output_validator
        def validate_final(ctx: RunContext[None], output: FinalSummary) -> FinalSummary:
            if not 5 <= len(output.key_points) <= 8:
                raise ModelRetry(
                    f'核心要点 {len(output.key_points)} 条不符合要求，需要 5-8 条'
                )
            if len(output.overview) < 80:
                raise ModelRetry(f'总览只有 {len(output.overview)} 字，需要 100-150 字')
            if len(output.title) > 30:
                raise ModelRetry(f'标题 {len(output.title)} 字太长，压缩到 30 字以内')
            if len(output.keywords) < 3:
                raise ModelRetry('关键词少于 3 个，请补充')
            return output

    async def summarize(self, text: str) -> SummarizeReport:
        """执行完整的 map-reduce 摘要流程"""
        self._retry_count = 0

        chunks = split_by_tokens(text, chunk_tokens=self.chunk_tokens)
        logger.info(
            f'原文约 {count_tokens(text)} tokens，分成 {len(chunks)} 块，'
            f'并发上限 {self.max_concurrent}'
        )

        chunk_points, failed = await self._run_map(chunks)

        if not chunk_points:
            raise RuntimeError(f'全部 {len(chunks)} 个分块都处理失败')

        logger.info(f'Map 完成：{len(chunk_points)} 条要点，失败 {failed} 块')

        final = await self._run_reduce(chunk_points)

        return SummarizeReport(
            summary=final,
            chunk_total=len(chunks),
            chunk_failed=failed,
            map_retries=self._retry_count,
        )

    async def _run_map(self, chunks: list[str]) -> tuple[list[str], int]:
        """Map 阶段：并发处理所有分块，返回 (所有要点, 失败块数)"""
        sem = asyncio.Semaphore(self.max_concurrent)

        async def run_one(idx: int, chunk: str) -> list[str] | None:
            async with sem:
                try:
                    result = await self.map_agent.run(
                        f'提炼以下片段的要点：\n\n{chunk}'
                    )
                    logger.debug(f'块 {idx + 1}/{len(chunks)} 完成')
                    return result.output.points
                except UnexpectedModelBehavior as e:
                    logger.warning(f'块 {idx + 1} 重试用尽，跳过：{e}')
                    return None
                except Exception as e:
                    logger.warning(f'块 {idx + 1} 异常，跳过：{e}')
                    return None

        results = await asyncio.gather(
            *[run_one(i, c) for i, c in enumerate(chunks)]
        )

        all_points: list[str] = []
        failed = 0
        for r in results:
            if r is None:
                failed += 1
            else:
                all_points.extend(r)

        return all_points, failed

    async def _run_reduce(self, points: list[str]) -> FinalSummary:
        """Reduce 阶段：合并要点。要点过多时先做一轮预压缩"""
        # 要点太多可能撑爆 reduce 的上下文，先分组压缩一轮
        if len(points) > 60:
            logger.info(f'要点过多（{len(points)}），先做一轮预压缩')
            points = await self._pre_reduce(points)

        joined = '\n'.join(f'- {p}' for p in points)
        result = await self.reduce_agent.run(
            f'把以下 {len(points)} 条要点合并成一份完整摘要：\n\n{joined}'
        )
        return result.output

    async def _pre_reduce(self, points: list[str], group_size: int = 30) -> list[str]:
        """要点分组预压缩，避免 reduce 阶段上下文过长"""
        groups = [points[i:i + group_size] for i in range(0, len(points), group_size)]

        async def compress(group: list[str]) -> list[str]:
            joined = '\n'.join(f'- {p}' for p in group)
            result = await self.map_agent.run(
                f'把以下要点合并去重，压缩成 3-4 条：\n\n{joined}'
            )
            return result.output.points

        compressed = await asyncio.gather(*[compress(g) for g in groups])
        return [p for group in compressed for p in group]
```

### 5.5 CLI 入口 `cli.py`

```python
import asyncio
import logging
from pathlib import Path

import click
from dotenv import load_dotenv

from src.summarizer import LongTextSummarizer
from src.splitter import count_tokens

load_dotenv()
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
)


@click.command()
@click.argument('input_file', type=click.Path(exists=True, path_type=Path))
@click.option('--output', '-o', type=click.Path(path_type=Path), help='输出文件，默认打印到终端')
@click.option('--map-model', default='openai:gpt-4o-mini', help='Map 阶段模型')
@click.option('--reduce-model', default='openai:gpt-4o', help='Reduce 阶段模型')
@click.option('--chunk-tokens', default=1500, help='每块 token 数')
@click.option('--concurrent', '-c', default=5, help='最大并发数')
def main(input_file, output, map_model, reduce_model, chunk_tokens, concurrent):
    """长文本摘要工具"""
    text = input_file.read_text(encoding='utf-8')
    click.echo(f'读取 {input_file}，约 {count_tokens(text)} tokens')

    summarizer = LongTextSummarizer(
        map_model=map_model,
        reduce_model=reduce_model,
        chunk_tokens=chunk_tokens,
        max_concurrent=concurrent,
    )

    try:
        report = asyncio.run(summarizer.summarize(text))
    except RuntimeError as e:
        click.echo(click.style(f'处理失败：{e}', fg='red'), err=True)
        raise SystemExit(1)

    s = report.summary
    lines = [
        f'# {s.title}',
        '',
        s.overview,
        '',
        '## 核心要点',
        *[f'{i}. {p}' for i, p in enumerate(s.key_points, 1)],
        '',
        f'关键词：{", ".join(s.keywords)}',
    ]
    content = '\n'.join(lines)

    if output:
        output.write_text(content, encoding='utf-8')
        click.echo(click.style(f'已写入 {output}', fg='green'))
    else:
        click.echo('\n' + content)

    click.echo(
        f'\n分块 {report.chunk_total} 个'
        f'（失败 {report.chunk_failed}），Map 最大重试 {report.map_retries} 次'
    )


if __name__ == '__main__':
    main()
```

**使用示例**：
```bash
# 基础用法
python cli.py report.txt

# 输出到文件，提高并发
python cli.py contract.txt -o summary.md --concurrent 10

# 省钱模式：两个阶段都用便宜模型
python cli.py report.txt --map-model openai:gpt-4o-mini --reduce-model openai:gpt-4o-mini
```

### 5.6 依赖文件 `requirements.txt`

```txt
pydantic-ai==0.0.14
pydantic==2.10.4
openai==1.57.4
langchain-text-splitters==0.3.4
tiktoken==0.8.0
python-dotenv==1.0.1
click==8.1.8
```

---

## 6. 常见坑与易错点

### 坑点 1：校验器里 `raise ValueError` 不会触发重试

**问题**：
```python
@agent.output_validator
def validate(output: Summary) -> Summary:
    if len(output.key_points) < 3:
        raise ValueError('要点不足')   # ❌ 直接抛给调用方，不会重试
    return output
```

**解决**：必须用 `ModelRetry`
```python
from pydantic_ai import ModelRetry

@agent.output_validator
def validate(output: Summary) -> Summary:
    if len(output.key_points) < 3:
        raise ModelRetry('要点不足 3 条，请补充')   # ✅ 框架带着原因重试
    return output
```

**区分标准**：模型重做一次有可能修好 → `ModelRetry`；配置错、网络断等模型改不了的 → 普通异常。

---

### 坑点 2：`ModelRetry` 消息太笼统，模型改不对

**问题**：
```python
raise ModelRetry('格式错误')   # ❌ 模型不知道错在哪，第二次大概率还错
```

**解决**：把具体差距和期望写进去
```python
raise ModelRetry(
    f'key_points 只有 {len(output.key_points)} 条，需要 5-8 条。'
    f'请从原文中补充遗漏的要点，每条不超过 50 字。'
)
```

`ModelRetry` 的消息是直接发给模型的，写得越具体，修对的概率越高。

---

### 坑点 3：重试次数用尽会抛 `UnexpectedModelBehavior`，没兜底就整个任务挂掉

**问题**：
```python
# ❌ 批量处理 100 块，第 7 块重试用尽 → 整个 gather 抛异常，前面 6 块的结果也白跑
results = await asyncio.gather(*[agent.run(c) for c in chunks])
```

**解决**：单块兜底 + 记录失败
```python
from pydantic_ai.exceptions import UnexpectedModelBehavior

async def run_one(chunk):
    try:
        return (await agent.run(chunk)).output
    except UnexpectedModelBehavior as e:
        logger.warning(f'重试用尽，跳过该块：{e}')
        return None

results = await asyncio.gather(*[run_one(c) for c in chunks])
valid = [r for r in results if r is not None]
```

也可以用 `asyncio.gather(..., return_exceptions=True)`，但自己 try 更容易区分错误类型。

---

### 坑点 4：嵌套模型 + 严格校验导致重试次数被快速烧光

**问题**：
```python
class Item(BaseModel):
    name: str
    score: float = Field(ge=0, le=1)   # 严格区间

class Result(BaseModel):
    items: list[Item] = Field(min_length=10)   # 至少 10 个
```
10 个 item 里只要 1 个 `score` 越界，整次输出就失败。重试 3 次可能都撞在不同 item 上。

**解决**：放宽 schema，在校验器里做修正而不是拒绝
```python
class Item(BaseModel):
    name: str
    score: float   # 不在 schema 层设区间

@agent.output_validator
def fix_scores(output: Result) -> Result:
    # 能修的就修，不要动不动就 ModelRetry
    for item in output.items:
        item.score = min(max(item.score, 0.0), 1.0)

    if len(output.items) < 10:
        raise ModelRetry(f'只给了 {len(output.items)} 项，需要至少 10 项')
    return output
```

**原则**：schema 只管结构，能程序化修正的问题在校验器里修，只有必须模型重做的才 `ModelRetry`。

---

### 坑点 5：分块没有重叠，跨块信息被切断

**问题**：
```python
splitter = RecursiveCharacterTextSplitter(chunk_size=3000, chunk_overlap=0)
# ❌ "本合同自 2026 年 1 月 1 日起生效，有效期" | "三年" 被切成两块
# 两块各自摘要都得不到完整信息
```

**解决**：留 5%-10% 的重叠
```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=3000,
    chunk_overlap=200,       # 约 7%
    separators=['\n\n', '\n', '。', '！', '？', ' ', ''],
)
```

重叠会增加约 7% 的 token 成本，但换来跨块语义完整，值得。

---

### 坑点 6：中文文本用默认分隔符，块大小忽大忽小

**问题**：
```python
# ❌ 默认 separators 是 ['\n\n', '\n', ' ', '']，中文没有空格分词
# 会一路 fallback 到按字符硬切，切在词中间
splitter = RecursiveCharacterTextSplitter(chunk_size=3000, chunk_overlap=200)
```

**解决**：显式加中文标点
```python
separators = ['\n\n', '\n', '。', '！', '？', '；', '，', ' ', '']
```

顺序很重要 — 从最"粗"的分隔符往细的排，splitter 会优先在粗粒度处切。

---

### 坑点 7：Reduce 阶段上下文超限

**问题**：
```python
# ❌ 100 个块 × 每块 4 条要点 = 400 条要点，一次性丢给 reduce
# 可能超出上下文，或者模型"看不完"导致漏信息
joined = '\n'.join(all_points)
await reduce_agent.run(f'合并：{joined}')
```

**解决**：分层 reduce（示例 5.4 的 `_pre_reduce`）
```python
if len(points) > 60:
    points = await self._pre_reduce(points)   # 先分组压缩一轮
```

要点超过一定数量就先做一轮"要点压缩要点"，再进最终 reduce。

---

### 坑点 8：`result.data` 已废弃

**问题**：
```python
result = agent.run_sync(prompt)
print(result.data)   # ⚠️ 旧 API，新版本会有 deprecation 警告
```

**解决**：
```python
print(result.output)   # ✅ 新 API
```

同理，`result_type` → `output_type`，`@agent.result_validator` → `@agent.output_validator`。写新代码直接用新名字。

---

## 7. 验证你学会了（自测题）

### 练习 1：让校验器在最后一次重试时放宽标准

**需求**：  
要点不足 5 条时正常 `ModelRetry`，但如果已经是最后一次重试机会，就接受现有结果（避免整块失败）。

**提示**：`RunContext` 有 `ctx.retry` 属性表示当前重试次数。

<details>
<summary>参考答案</summary>

```python
from pydantic_ai import Agent, ModelRetry, RunContext

MAX_RETRIES = 3

agent = Agent('openai:gpt-4o-mini', output_type=FinalSummary, retries=MAX_RETRIES)


@agent.output_validator
def validate_lenient_at_end(ctx: RunContext[None], output: FinalSummary) -> FinalSummary:
    if len(output.key_points) >= 5:
        return output

    # 最后一次机会：不再重试，接受但记录
    if ctx.retry >= MAX_RETRIES - 1:
        logger.warning(f'最后一次重试，接受 {len(output.key_points)} 条要点')
        return output

    raise ModelRetry(f'要点只有 {len(output.key_points)} 条，需要 5-8 条')
```

关键是用 `ctx.retry` 判断还剩几次机会，避免"宁死不屈"导致整块数据丢失。

</details>

---

### 练习 2：实现带成本估算的分块预检

**需求**：  
在真正调用模型前，先算出这次任务大概要花多少钱，超过阈值就让用户确认。

**提示**：`count_tokens` 算输入 token，按模型单价估算；别忘了 Map 和 Reduce 两个阶段。

<details>
<summary>参考答案</summary>

```python
# 单价：美元 / 1M tokens（输入价，示意值，实际以官方价目为准）
PRICING = {
    'openai:gpt-4o-mini': 0.15,
    'openai:gpt-4o': 2.50,
}


def estimate_cost(
    text: str,
    map_model: str,
    reduce_model: str,
    chunk_tokens: int = 1500,
) -> dict:
    chunks = split_by_tokens(text, chunk_tokens=chunk_tokens)
    total_input = count_tokens(text)

    # Map：输入约等于原文（含重叠，加 10%）
    map_tokens = int(total_input * 1.1)
    map_cost = map_tokens / 1_000_000 * PRICING[map_model]

    # Reduce：输入约等于所有要点，粗估每块产出 200 tokens
    reduce_tokens = len(chunks) * 200
    reduce_cost = reduce_tokens / 1_000_000 * PRICING[reduce_model]

    return {
        'chunks': len(chunks),
        'map_tokens': map_tokens,
        'reduce_tokens': reduce_tokens,
        'total_cost_usd': round(map_cost + reduce_cost, 4),
    }


# CLI 里加确认
est = estimate_cost(text, 'openai:gpt-4o-mini', 'openai:gpt-4o')
click.echo(f"预计 {est['chunks']} 块，成本约 ${est['total_cost_usd']}")
if est['total_cost_usd'] > 1.0:
    click.confirm('成本超过 $1，继续？', abort=True)
```

注意这只是输入侧估算，输出 token 单价通常更高，实际成本会略高于估算值。

</details>

---

### 练习 3：给 map 阶段加"上一块的尾部上下文"

**需求**：  
除了字符重叠，再给每个块附上前一块摘要的最后一条要点，让模型知道上文讲了什么，减少割裂感。

**提示**：这会让 map 阶段变成部分串行，需要权衡速度。

<details>
<summary>参考答案</summary>

```python
async def run_map_with_context(self, chunks: list[str]) -> list[str]:
    """串行 map，每块携带上一块的尾部要点作为上下文"""
    all_points: list[str] = []
    prev_tail = ''

    for idx, chunk in enumerate(chunks):
        prompt = (
            f'上文最后提到：{prev_tail}\n\n' if prev_tail else ''
        ) + f'提炼以下片段的要点：\n\n{chunk}'

        try:
            result = await self.map_agent.run(prompt)
            points = result.output.points
            all_points.extend(points)
            prev_tail = points[-1] if points else ''
        except Exception as e:
            logger.warning(f'块 {idx + 1} 失败：{e}')
            prev_tail = ''

    return all_points
```

**权衡**：串行让 100 块的处理时间从"并发的 1/5"变成完整的 100 次串行，慢很多。折中做法是分段并发 — 每 5 块一组，组内并发、组间传递上下文。

</details>

---

## 总结

本教程展示了如何用 Pydantic AI 处理长文本并保证输出质量。核心要点：

1. **结构化输出**：`output_type=YourModel`，解析和格式重试框架全包
2. **业务校验**：`@agent.output_validator` + `raise ModelRetry('具体原因')`，不用手写重试循环
3. **重试上限**：`retries=N`，用尽后抛 `UnexpectedModelBehavior`，批量场景必须单点兜底
4. **分块要自己做**：`langchain-text-splitters` + 中文分隔符 + 5%-10% 重叠
5. **Map-Reduce**：Map 用便宜模型跑并发，Reduce 用好模型保质量，要点过多先预压缩

与原生 API 相比，Pydantic AI 省掉的是"解析 + 校验 + 拼重试提示词 + 数重试次数"这一整套样板代码（通常 40-60 行降到 5-10 行）；不省的是分块策略和 map-reduce 编排，这部分逻辑仍需自己设计。

**下一步学习**：
- 阅读 [AI应用安全与交付形态-Pydantic-AI版](./AI应用安全与交付形态-Pydantic-AI版.md)
- 回顾 [并发批处理与缓存层-Pydantic-AI版](./并发批处理与缓存层-Pydantic-AI版.md)
