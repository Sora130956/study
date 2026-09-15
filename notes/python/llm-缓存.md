# LLM 应用的缓存：什么能缓存、键怎么算、流式怎么办

> **目标读者**：会用 httpx / asyncio 调 LLM API，想给 AI 应用加缓存省钱提速，但卡在「用户输入的是一句自由 prompt，这玩意儿怎么可能命中」的人。
> **前置知识**：[asyncio-入门](./asyncio-入门.md)、[sse-流式响应](./sse-流式响应.md)；TTL / 命中 / read-through 基本概念（不熟悉可先读标准库 `functools.lru_cache` 文档）。
> **学习时长**：约 1.5 小时。第 2 节是本文的核心判断依据，别跳；第 6 节（语义缓存）可先只读结论。

***

## 1. 先澄清：你有两层缓存，它们不互相替代

<mark style="background: #BBFABBA6;">新手最常见的误解是「OpenAI / Anthropic 自己不是有 prompt caching 吗，我还缓存什么」。这两个东西缓存的**根本不是同一样东西**。</mark>

| <br />  | 提供商的 Prompt Caching                    | 你的应用层缓存             |
| ------- | -------------------------------------- | ------------------- |
| 缓存什么    | 输入前缀的 KV 状态（模型内部的中间计算）                 | **最终生成的完整回答文本**     |
| 命中时省什么  | input token 打 1 折                      | **全省**：0 token、0 等待 |
| 命中时还生成吗 | **照样生成**，output token 全价，延迟一秒不少        | 不生成，直接返回字符串         |
| 命中条件    | 前缀逐字节相同，且超过最小阈值（各家 1024～4096 token 不等） | 你自己定义的 cache key 相同 |
| 谁控制     | 提供商（部分需要你打 `cache_control` 标记）         | 你                   |

一句话记住：**Prompt caching 优化的是「同一份长 system prompt / 长文档被反复用」时的单次调用成本；应用层缓存优化的是「这次调用干脆别发生」。**

所以生产里两个都开：应用层缓存吃掉重复请求，剩下真正要打出去的请求靠 prompt caching 降成本。

> **一个容易忽略的事实**：prompt caching 对 output token 一分钱不省。如果你的场景是「短输入、长输出」（比如写文案、生成报告），提供商的缓存几乎帮不上忙，能救你的只有应用层缓存。

***

## 2. 核心判断：哪些 LLM 调用能缓存

<mark style="background: #BBFABBA6;">你的直觉是对的 —— **用户手打的一句自由 prompt，本来就不该指望精确命中**。但 AI 应用里的 LLM 调用远不止这一种，而且**可缓存的那些往往是调用量的大头**。</mark>

| 调用类型                 | 输入来自哪          | 能缓存吗                     | 键怎么算     |
| -------------------- | -------------- | ------------------------ | -------- |
| 内部管线：打标签、抽字段、翻译、生成摘要 | 程序拼的模板 + 有限的数据 | **能，命中率很高**              | 数据哈希     |
| Embedding            | 文本片段           | **能，最该缓存**（同一段文本的向量永远相同） | 文本哈希     |
| RAG 检索 / rerank      | query          | **能**                    | query 哈希 |
| 预设问题、FAQ、示例按钮        | 固定集合           | **能**                    | 问题 id    |
| 用户自由聊天               | 人手打            | **基本不能**（除非上语义缓存，见第 6 节） | —        |

**举个具体例子**：给一条 job description 生成「技能标签 + 难度评分」。这个调用的变量部分就是 job description 本身 —— 同一条 job 被不同用户打开、被重新跑一遍分析、被批量任务扫到，**输入完全一样，用户根本没在这一步打字**。这就是天然的精确命中，而且它可能占你 LLM 调用量的 80%。

**所以正确的上手顺序是**：

1. 先把**内部确定性调用**全缓存了 —— 覆盖大头，零风险，改动小。
2. Embedding 单独缓存 —— 同一段文本反复算向量是纯浪费。
3. 自由对话那一路**先不缓存**，等你真的有量了再单独讨论语义缓存。

***

## 3. 缓存键：不是只哈希 prompt 文本

**<mark style="background: #BBFABBA6;">缓存键必须包含所有影响输出的东西。</mark>** 漏掉任何一个，就是一次线上事故。

```python
import hashlib

PROMPT_VERSION = "v3"          # 模板每改一次，手动 +1（或用模板文件的哈希自动算）

def cache_key(
    *,
    model: str,                # 换模型 = 换答案
    prompt_version: str,       # 改模板 = 换答案（最容易漏！）
    temperature: float,        # 采样参数影响输出
    user_input: str,
) -> str:
    # 归一化：大小写、首尾空白、中间连续空白统一，让「等价输入」真的能撞上同一个 key
    normalized = " ".join(user_input.lower().split())
    raw = f"{model}|{prompt_version}|{temperature}|{normalized}"
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()
```

**三个必须放进 key 的东西**：

- <mark style="background: #BBFABBA6;">**`prompt_version`（最容易漏）**：你改了 system prompt 想让输出变好，结果线上全是缓存里的旧答案，怎么测都不生效，排查半天。**只要模板变了，key 就必须变。** 推荐做法是直接把模板文本的哈希当 version，改了自动失效，不依赖人记得手动加。</mark>
- <mark style="background: #BBFABBA6;">**`model`**：从 haiku 换成 sonnet，答案质量完全不同，绝不能共用缓存。</mark>
- **`temperature`**：见下条。

**<mark style="background: #BBFABBA6;">关于** **`temperature > 0`** **的取舍</mark>**：temperature 大于 0 的设计意图就是「同样输入给不同答案」，加缓存等于把它冻死成固定答案。两种处理方式，选一个并写进注释：

- 这类调用**不缓存**（创意生成、多样化推荐属于这类）；
- 或者明确接受「同输入同答案」，把 temperature 放进 key，保证至少不同参数不串味。

**还有一条在 LLM 场景特别致命的**：

> **<mark style="background: #BBFABBA6;">凡是 prompt 里拼了用户私有数据的，key 必须带** **`user_id`，或者干脆不缓存。</mark>**

因为 LLM 的 prompt 经常是「系统模板 + 用户的简历 / 聊天记录 / 订单」拼出来的。<mark style="background: #BBFABBA6;">如果 key 只哈希了「用户那句问题」而没带上下文里的私有数据，A 用户会直接拿到 B 用户的答案。这不是性能问题，是数据泄露事故。</mark>

***

## 4. 代码：从最简单的开始

### 4.1 内部确定性调用 —— 一个装饰器搞定

不要手写缓存类。`async-lru` 已经把 TTL、LRU 淘汰、并发单飞（同一个 key 并发只真正调一次）全做完了。

```python
from async_lru import alru_cache

@alru_cache(maxsize=1024, ttl=3600)
async def extract_skills(job_description: str) -> list[str]:
    """从 job description 抽技能标签。同一条 JD 一小时内只调一次 LLM。"""
    resp = await client.post("/v1/chat", json={
        "model": MODEL,
        "messages": [
            {"role": "system", "content": SKILL_PROMPT},
            {"role": "user", "content": job_description},
        ],
    })
    resp.raise_for_status()
    return parse_skills(resp.json())
```

**注意这里的隐患**：`SKILL_PROMPT` 和 `MODEL` 是外部常量，没进 key。你改了 `SKILL_PROMPT`，缓存不会失效。所以装饰器写法只适合「模板基本不动」的稳定调用；模板还在频繁迭代的阶段，用下面 4.2 的显式写法。

### 4.2 显式 key —— 模板迭代期该用这个

```python
import hashlib
from async_lru import alru_cache

SKILL_PROMPT = "..."
# 用模板内容的哈希当版本号：改一个字，key 自动全变，旧缓存自然失效
PROMPT_HASH = hashlib.sha256(SKILL_PROMPT.encode()).hexdigest()[:8]

@alru_cache(maxsize=1024, ttl=3600)
async def _extract_cached(key: str, job_description: str) -> list[str]:
    """真正调 LLM。key 只用于参与缓存匹配，函数体里不使用它。"""
    ...  # 同上

async def extract_skills(job_description: str) -> list[str]:
    key = f"{MODEL}|{PROMPT_HASH}"
    return await _extract_cached(key, job_description)
```

把「影响输出但不是业务参数」的东西（model、模板版本）拼成一个 `key` 参数传进去，`alru_cache` 会把所有参数一起算进缓存键。模板一改，`PROMPT_HASH` 变，全部旧缓存自动失效。

### 4.3 Embedding 缓存 —— 收益最高、风险最低

```python
@alru_cache(maxsize=10000, ttl=86400)
async def embed(text: str) -> list[float]:
    """同一段文本的向量是确定的，TTL 可以给很长。"""
    resp = await client.post("/v1/embeddings", json={"model": EMBED_MODEL, "input": text})
    return resp.json()["data"][0]["embedding"]
```

Embedding 是**纯函数**：同样的文本 + 同样的模型 = 完全相同的向量，永远不会「过时」。TTL 给 24 小时甚至更长都行，这里 TTL 的作用只是配合 LRU 控制内存，不是担心数据陈旧。

***

## 5. 流式（SSE）场景的两个坑

这两个坑跟 [sse-流式响应](./sse-流式响应.md) 直接相关，也是加缓存时最容易翻车的地方。

### 坑 A：命中缓存时，不能一次性把整段吐出去

你缓存的是一段**完整文本**，但接口是流式的。命中时如果一次 `yield` 整段，前端 `EventSource` 会看到「等了 0 秒，然后瞬间出现一大坨」，和未命中时逐字蹦出来的体验完全割裂，用户会以为出 bug 了。

做法是命中后重新切片，加一个极小的间隔模拟成流：

```python
async def stream_from_cache(text: str):
    """命中缓存时，把完整文本重新切成小块推送，保持和真实流式一致的观感。"""
    for i in range(0, len(text), 20):
        yield f"data: {text[i:i + 20]}\n\n"
        await asyncio.sleep(0.01)   # 间隔要小，别为了"像"而故意拖慢
```

### 坑 B：流没走完，绝不能写缓存（更致命）

流式生成过程中，客户端可能断连、上游可能中途报错、超时可能触发。这时候你手里是**半截答案**。如果按「边收边攒，收完就存」的朴素写法，很容易把这半截存进缓存 —— **后面所有命中这个 key 的用户，拿到的都是这个残缺答案**，而且因为它"命中"了，连重试的机会都没有。

必须确保只有在生成器**正常走完**的路径上才写缓存：

```python
async def stream_llm(key: str, prompt: str):
    chunks: list[str] = []
    async for chunk in call_llm_stream(prompt):
        chunks.append(chunk)
        yield f"data: {chunk}\n\n"
    # 只有整个 async for 正常结束（没抛异常、没被 GeneratorExit 打断）才会走到这里
    cache[key] = CacheEntry("".join(chunks), ttl=3600)
```

注意**不要**把写缓存放进 `finally`：`finally` 在客户端断连（`GeneratorExit`）和上游异常时同样会执行，那就正好把残缺答案存进去了，完全违背初衷。

> **和直觉相反的一点**：加了 `try/finally` 通常是「更健壮」的写法，但在这里恰恰是 bug 的来源。缓存写入必须在**成功路径**上，不是在**清理路径**上。

***

## 6. 语义缓存：想缓存自由 prompt 的唯一出路，但慎用

**思路**：把用户问题转成 embedding，去向量库里找相似度超过阈值的历史问题，命中就返回当时的答案。现成方案有 GPTCache。

**致命问题是「语义相似 ≠ 答案等价」**：

| 两个问题                        | embedding 相似度   | 答案        |
| --------------------------- | --------------- | --------- |
| 「北京今天天气」/「北京明天天气」           | 极高              | **完全不同**  |
| 「这个方案有什么优点」/「这个方案有什么缺点」     | 极高（反义词在向量空间里很近） | **相反**    |
| 「订单 12345 状态」/「订单 12346 状态」 | 几乎是 1           | **不同的订单** |

embedding 对**数字、日期、实体名、否定词**这些「决定答案的关键差异」恰恰最不敏感。

于是阈值怎么调都是错：

- 调高 → 几乎不命中，白上一套向量库 + 每次还多一次 embedding 调用，**比不缓存更慢更贵**；
- 调低 → 给用户返回**错误答案**，比不缓存糟糕一百倍。

**结论**：学习阶段和大部分生产场景都**不要碰语义缓存**。真要做的前提是三个都满足：① 场景窄（FAQ 类、答案本来就宽泛）；② 有人工标注的评测集能量化「误命中率」；③ 误命中的代价可接受。

***

## 7. 常见坑速查

**坑 1：改了 prompt 模板但缓存没失效**

- 现象：改完 system prompt 部署上去，输出一点没变，反复怀疑自己没部署成功。
- 原因：cache key 里没有模板版本。
- 解决：把模板文本的哈希放进 key（见 4.2），改了自动失效。

**坑 2：缓存串到了别的用户**

- 现象：A 用户看到 B 用户的简历分析结果。
- 原因：prompt 里拼了用户私有数据，但 key 只哈希了那句问题。
- 解决：key 带 `user_id`，或这类调用不缓存。

**坑 3：把半截流式答案写进了缓存**

- 现象：某个问题永远返回一个被截断的答案，且刷新也没用。
- 原因：写缓存的代码放在 `finally` 里，或没区分正常结束与异常中断。
- 解决：见第 5 节坑 B，只在成功路径写。

**坑 4：缓存了带工具调用 / 副作用的响应**

- 现象：LLM 返回的 tool\_call 被缓存，导致「该查的没查」或「该下的单下了两次」。
- 原因：把「决定要调用工具」的那次响应当成纯文本缓存了。
- 解决：只缓存**最终的纯文本回答**。任何会触发外部动作的中间响应不缓存。

**坑 5：以为开了 prompt caching 就不用应用层缓存**

- 现象：账单没怎么降，延迟一点没改善。
- 原因：prompt caching 只打折 input token，output 全价、延迟照旧。
- 解决：见第 1 节，两层是互补关系。

**坑 6：给高 temperature 的创意生成加了缓存**

- 现象：用户点「换一个」，返回的还是同一段文案。
- 原因：缓存把随机性冻死了。
- 解决：这类调用不缓存；或者让用户的「换一个」动作往 key 里加一个递增的 nonce。

***

## 8. 验证你学会了

1. **复述**：提供商的 prompt caching 和你自己的应用层缓存，各自缓存什么？命中时分别省了什么？（答案在第 1 节）
2. **判断**：下面四个调用哪些该缓存、key 里要放什么？
   ① 把用户上传的 PDF 切块后逐块生成 embedding；
   ② 用户在聊天框里随便问一句；
   ③ 给一条固定的商品描述生成营销文案，temperature=0.9；
   ④ 对用户的简历和某个岗位做匹配度打分。
   （答案在第 2、3 节）
3. **组合应用**：给一个流式接口 `async def chat_stream(user_id: str, question: str)` 加缓存，要求：① key 正确包含 model、模板版本、user\_id；② 命中时仍然以流的形式返回；③ 生成中断时不写缓存。（答案在第 3、5 节）

