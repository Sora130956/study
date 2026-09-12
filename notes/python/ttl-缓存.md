# TTL 缓存入门：相同输入命中缓存，省掉重复的 LLM / 网络调用

- 目标读者：会 Python + asyncio + httpx，知道 `await`、`AsyncClient` 是什么
- 前置知识：第 3 节的「缓存、TTL、命中/未命中、read-through、单调时钟」会从零讲清
- 学习时长：约 40 分钟 / 可跳过：第 2 节是反面推演，急着写代码可先跳到第 4 节

> 教学锚点：本文示例是对工作区 `freelancer-job-analyzer/backend/services/api_client.py` 里真实生产代码（`CacheEntry` 类 + `fetch_jobs`/`fetch_currencies` 的 read-through 模式）的教学简化，不是凭空造的例子。

---

## 第 1 节：它是什么（一句话 + 一句类比）

**一句话**：TTL 缓存 = 把一次昂贵的计算结果存起来，并给它设一个「保质期」；保质期内再来相同的输入，直接拿存好的结果不再重算；过了保质期就重新算、重新存。

这里的「昂贵」，对你正在做的 AI 应用来说，主要是两样：**调一次 LLM 要花钱**（token 计费），**调一次外部 API 要时间**（网络往返）。

**类比：冰箱里现成的菜**

你晚上要做饭，先打开冰箱看一眼：

- 如果昨天做多了的菜**还在保质期内**，直接热一下吃 —— 这就是「命中缓存」，不出门、不花钱、秒好。
- 如果冰箱里没有，或者菜**过期了**（过了 TTL），就得出门买菜重新做 —— 这就是「未命中」，要付全价、要等。

缓存不是「永久记一次」。菜放久了会坏，所以每样东西都得贴一个「保质期标签」—— 这个保质期就是 **TTL（Time To Live，存活时间）**。过了标签上的时间，就得重新买。

这个类比后面会真实对应到代码：

| 类比里的东西 | 代码里的东西 |
| --- | --- |
| 冰箱 | 一个存结果的字典 / 缓存结构 |
| 菜 | 一次昂贵调用的结果（LLM 回复、API 响应） |
| 保质期标签 | `expires_at = time.monotonic() + ttl` |
| 看一眼没过期就吃 | `if cache.is_valid: return cache.value` |
| 过期了重新买 | 没命中就调上游、算结果、存进缓存 |

---

## 第 2 节：它解决什么问题

### 2.1 先看不用它会怎样

场景：你写了一个「给产品写一句话简介」的函数，背后调 LLM。同一个产品 ID 会被请求很多次：

```python
async def get_summary(product_id: str) -> str:
    # 每次调用都真的发一次 LLM 请求 —— 每次都要花 token 钱、等几秒
    resp = await client.post("/v1/chat", json={"prompt": f"简介 {product_id}"})
    return resp.json()["choices"][0]["message"]["content"]
```

**可感知的痛点**（不是抽象的「不好维护」，是具体的）：

1. **相同问题反复付全价**：同一个 `product_id` 被 100 个用户各问一次，你付了 100 次 LLM 的钱，答案还是同一个。
2. **慢**：100 次调用哪怕缓存能省 90 次，也省掉了 90 次网络往返的等待。
3. **可能触发限流**：短时间内对上游打太多次，还没缓存就被 429 限流了。

### 2.2 那用个 dict 永久缓存不就行了？

新手直觉是「那我用一个字典存下来，下次直接查」：

```python
cache: dict[str, str] = {}          # 永久缓存：只进不出

async def get_summary(product_id: str) -> str:
    if product_id in cache:          # 命中就直接返回
        return cache[product_id]
    resp = await client.post(...)
    result = resp.json()["choices"][0]["message"]["content"]
    cache[product_id] = result       # 存进去，再也不清理
    return result
```

这个朴素做法能用，但有两个坑：

- **数据永远不更新**：产品信息改了，简介还是旧的。缓存里永远是「第一次算出来的那份」。
- **内存只涨不降**：product_id 有多少个，字典就有多少条，永不淘汰。跑得久一点内存就慢慢吃满。

### 2.3 问题根源（一句话）

**根源是「缓存没有过期概念」—— 要么永久缓存（数据陈旧 + 内存泄漏），要么不缓存（重复付全价）。** TTL 缓存补上「保质期」，两头都解决：数据会过期重算，不会陈旧；过期条目可以被清掉，内存不会无限涨。

### 2.4 经典应用场景

- 调 LLM / 外部 API，结果在几分钟内不会变：缓存相同输入。
- 汇率、技能标签这类「一天才变一次」的数据：给一天的长 TTL（工作区真实代码里 `CURRENCIES_CACHE_TTL = 24 * 60 * 60`）。
- 数据库查询结果、RAG 检索结果缓存。

---

## 第 3 节：读下去之前，先搞懂这些概念

| 概念 | 一行定义 | 一句大白话 |
| --- | --- | --- |
| 缓存 cache | 一个「key → value」的存储，key 是输入，value 是结果 | 冰箱里贴着标签的菜 |
| TTL | Time To Live，条目存活的秒数 | 保质期有多长 |
| 命中 hit | 查缓存时 key 在且没过期 | 冰箱里有能吃的 |
| 未命中 miss | key 不在或已过期 | 得出门买 |
| read-through | 查不到就去上游取、取了再存、再返回的模式 | 先翻冰箱，没有再去买，买回来顺手放冰箱 |
| 缓存键 cache key | 用来查缓存的唯一标识，通常由输入归一化而来 | 菜名标签 |
| `time.monotonic()` | 单调时钟，只保证「一直往前走」的秒数 | 一个不会被调表影响的计时器 |
| `time.time()` | 墙上时钟，会被系统改时间影响 | 挂钟，能被人拨快拨慢 |

**为什么用 `time.monotonic()` 而不是 `time.time()`（这条很重要）**：判断「过了多久」要用单调时钟。`time.time()` 是「现在几点了」，如果系统时间被 NTP 校时或用户手动改了（比如夏令时、时区切换、对表），它可能**倒退**，于是「过期时间减当前时间」会算出负的或者超长的时间，缓存行为就乱了。`time.monotonic()` 专门用来测时间差，永远不会倒退。

最小示例（两个时钟的区别）：

```python
import time
a = time.monotonic()   # 单调钟：适合算「过了多久」
b = time.time()        # 墙上钟：适合记录「现在几点」
```

> 你多半已会用 `await` / `dict`，扫一眼上面这张表就可以直接进第 4 节。

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本 —— 一个会过期的缓存条目

先只做一个最小单位：一个「存了值 + 过期时间」的东西，能回答「我现在还有效吗」。这就是工作区真实代码里 `CacheEntry` 的教学简化版。

```python
import time

class CacheEntry:
    """一个缓存条目：存了值，以及它什么时候过期。"""
    def __init__(self, value, ttl: float):
        self.value = value                    # 缓存的结果本体
        # 存「绝对过期时间」而不是「存了多久」：过期时刻 = 现在 + 保质期
        self.expires_at = time.monotonic() + ttl   # 用单调时钟，原因见第 3 节

    @property
    def is_valid(self) -> bool:
        # 现在还没到过期时刻 → 还有效
        return time.monotonic() < self.expires_at

# --- 用起来 ---
entry = CacheEntry(value="结果 A", ttl=1.0)   # 保质期 1 秒
print(entry.is_valid)     # True：刚放进冰箱，当然能吃
time.sleep(1.2)           # 等 1.2 秒，超过保质期
print(entry.is_valid)     # False：过期了，得出门重买
```

**逐段解读**：

- `__init__` 里最关键的一行是 `self.expires_at = time.monotonic() + ttl`。它存的是「**未来某个时刻**」而不是「存了 5 秒」这种相对量。判断是否有效时，只需比较「现在」和「那个未来时刻」，一行搞定，不用记录创建时间再算差。
- `is_valid` 用了 `@property`，所以外面写 `entry.is_valid` 而不是 `entry.is_valid()`，读起来像「问它：你还新鲜吗」。
- 注意 `time.sleep(1.2)` 只是为了让示例能演示「过期」这个变化。真实代码里不会有 sleep，过期是**自然发生的**——随着时间流逝，`monotonic()` 越来越大，总会越过 `expires_at`。

**这一步你得到了什么**：一个能自己「到期」的最小缓存条目。判断有没有效，永远是一句 `is_valid`，不需要手工维护计时。

### 示例 2：真实需求 —— read-through 模式

现在把「先查缓存 → 命中直接返回 → 没命中才调上游 → 存缓存 → 返回」这条链串起来。这就是 `fetch_jobs` 的真实模式。

```python
import time
import httpx

class CacheEntry:                          # 复用示例 1 的定义
    def __init__(self, value, ttl: float):
        self.value = value
        self.expires_at = time.monotonic() + ttl

    @property
    def is_valid(self) -> bool:
        return time.monotonic() < self.expires_at

SKILLS_TTL = 60 * 60          # 技能标签 1 小时才变一次，给 1 小时保质期

class SkillClient:
    def __init__(self, client: httpx.AsyncClient):
        self._client = client
        self._cache: CacheEntry | None = None   # 只缓存一份「技能标签」结果

    async def fetch_skills(self) -> list[str]:
        # 第 1 步：先查缓存 —— 有且没过期，直接返回，不碰网络
        if self._cache and self._cache.is_valid:
            return self._cache.value

        # 第 2 步：未命中才发请求（真正的开销在这里，能被缓存省掉）
        resp = await self._client.get("/api/skills")
        resp.raise_for_status()
        skills = resp.json()["result"]

        # 第 3 步：算出来了，存进缓存，再返回
        self._cache = CacheEntry(value=skills, ttl=SKILLS_TTL)
        return skills
```

**逐段解读**：

- 判断命中的是 `if self._cache and self._cache.is_valid`：两个条件都得满足——`_cache` 不是 `None`（还没缓存过），并且它还没过期。第一次调用时 `_cache` 是 `None`，直接跳到发请求。
- **read-through 的精髓是「命中就完全绕开网络」**。第二次调用同一个 client 的 `fetch_skills()`，`_cache.is_valid` 为真，函数在第 1 步就 return 了，`await self._client.get(...)` 一行都没执行。
- 为什么把 `ttl` 提取成 `SKILLS_TTL` 常量而不是写死 `3600`：不同数据不同保质期，写在名字里，别人一眼看懂「技能标签 1 小时」。（真实代码里还有 `CURRENCIES_CACHE_TTL = 24 * 60 * 60`，汇率给 24 小时。）

**这一步你得到了什么**：一个能真正省钱的缓存函数——相同输入第二次开始，**零网络请求**，直接返回内存里的结果。

### 示例 3：贴近项目的样子 —— 缓存键 + 不同 TTL + 并发防击穿

真实场景里不会只有一个 `_cache` 字段存一个值，而是**一份字典，key 是输入，value 是 `CacheEntry`**，这样不同输入各有各的缓存。再补两个真实问题：怎么从输入生成 key，以及并发下多个协程同时未命中会不会重复调上游。

```python
import time
import asyncio
import hashlib
import httpx

class CacheEntry:
    def __init__(self, value, ttl: float):
        self.value = value
        self.expires_at = time.monotonic() + ttl

    @property
    def is_valid(self) -> bool:
        return time.monotonic() < self.expires_at

class SummaryService:
    def __init__(self, client: httpx.AsyncClient, ttl: float = 300.0):
        self._client = client
        self._ttl = ttl                          # 默认 5 分钟
        self._cache: dict[str, CacheEntry] = {}  # key -> 缓存条目
        self._locks: dict[str, asyncio.Lock] = {}  # 每个 key 一把锁，防重复请求

    def _key(self, text: str) -> str:
        # 用哈希把任意长度的输入压缩成定长 key：
        # 1) 避免 key 太长（长文本直接当 key 会又大又慢）
        # 2) 避免输入里有字典 key 不能用的字符
        return hashlib.sha256(text.encode("utf-8")).hexdigest()

    async def summarize(self, text: str) -> str:
        key = self._key(text)

        # 第 1 步：先查缓存
        entry = self._cache.get(key)
        if entry and entry.is_valid:
            return entry.value

        # 第 2 步：未命中，拿这把 key 专属的锁，避免并发时重复调上游
        lock = self._locks.setdefault(key, asyncio.Lock())
        async with lock:
            # 双重检查：可能等锁期间别的协程已经算好并放进缓存了
            entry = self._cache.get(key)
            if entry and entry.is_valid:
                return entry.value

            # 第 3 步：真正调 LLM（昂贵操作，只有第一个进来的协程会执行到这里）
            resp = await self._client.post("/v1/chat", json={"prompt": text})
            resp.raise_for_status()
            result = resp.json()["choices"][0]["message"]["content"]

            # 第 4 步：存缓存再返回
            self._cache[key] = CacheEntry(value=result, ttl=self._ttl)
            return result
```

**逐段解读**：

- `_key()` 用 `hashlib.sha256` 把输入压成定长哈希。为什么不全用原始文本当 key？① 一段 5000 字的文章直接当字典 key，字典内部要算哈希、存长串，又慢又占内存；② 统一成 64 个字符的十六进制串，干净、可控。代价是「相同哈希 ≠ 相同输入」在理论上可能碰撞，但 sha256 碰撞概率低到工程上可忽略。
- **缓存击穿**（`_locks` 那段）：假设 10 个请求同一瞬间都问同一个 `text`，都未命中，没有锁的话 10 个协程会**同时**去调 LLM——缓存完全没起到省钱作用，反而打得更凶。加一把 per-key 锁后，只有一个协程真正进去算，其余 9 个等锁；拿到锁后**再查一次缓存**（双重检查），发现已经有结果了，直接返回。
- 双重检查（第 2 步锁内再 `self._cache.get(key)` 一次）不是多余：等锁的协程醒来时，第一个协程可能已经把结果写进去了，不查就白调一次上游。
- 这个示例的缓存字典是**进程内**的：服务一重启就清空。真实生产里若要跨进程/跨实例共享，才需要 Redis 之类的进程外缓存（见第 5 节进阶）。

**生僻点展开：为什么「先查→取→存→返回」这个顺序不能乱**

read-through 的四步是有讲究的：

1. **必须先查再取**：如果不查就直接调上游，缓存就形同虚设。
2. **取到后必须存**：不存的话，下次还是未命中，等于没缓存。
3. **存完再返回**：存和返回之间没有网络，顺序无所谓，但「存」放返回前更自然——结果既然算出来了，先落袋再交给调用方。

**这一步你得到了什么**：一个贴近生产、能抗并发的 TTL 缓存服务——相同输入省掉重复调用，并发下不会重复付费。

## 第 5 节：最小可用的实际用法（生产场景模板）

一段可直接复制进真实项目的「缓存一个异步调用」helper。用它可以给任意一个 `async def` 取数函数套上 TTL 缓存。

```python
import time
import asyncio
from typing import Awaitable, Callable, Hashable

class TTLItem:
    def __init__(self, value, ttl: float):
        self.value = value
        self.expires_at = time.monotonic() + ttl
    @property
    def is_valid(self) -> bool:
        return time.monotonic() < self.expires_at

class TTLCache:
    def __init__(self):
        self._items: dict[Hashable, TTLItem] = {}          # [必须] 缓存本体
        self._locks: dict[Hashable, asyncio.Lock] = {}     # [可选] 防击穿用的锁

    async def get(
        self,
        key: Hashable,                                     # [必须] 缓存键
        fetch: Callable[[], Awaitable],                    # [必须] 未命中时的取数函数
        ttl: float,                                        # [必须] 保质期（秒）
    ):
        item = self._items.get(key)
        if item and item.is_valid:                          # [必须] 命中直接返回
            return item.value

        lock = self._locks.setdefault(key, asyncio.Lock())  # [可选] 并发防击穿
        async with lock:
            item = self._items.get(key)
            if item and item.is_valid:
                return item.value
            value = await fetch()                           # [必须] 真正调上游
            self._items[key] = TTLItem(value, ttl)          # [必须] 存缓存
            return value

# --- 用法 ---
cache = TTLCache()
async def fetch_price(symbol: str) -> float:
    return await cache.get(
        key=f"price:{symbol}",          # key 加个前缀，避免不同数据撞 key
        fetch=lambda: _real_fetch_price(symbol),
        ttl=60.0,                       # 价格 60 秒内不会大变
    )
```

**哪几行必须 / 哪些可删**：

- 必须：`_items` 字典、`is_valid` 命中判断、`await fetch()` 取数、`self._items[key] = TTLItem(...)` 存缓存——这四样缺一不可，构成 read-through 的最小闭环。
- 可选：`_locks` 那段锁 + 双重检查。如果你的服务是单协程顺序跑、或允许偶尔重复调一次上游，可以整段删掉，代码立刻短一半。
- `key=f"price:{symbol}"` 的前缀 `price:` 是个好习惯：不同业务的数据放同一个 cache 里，前缀避免 key 撞车。

**实际项目通常还配什么（进阶，可先跳过）**：

- **淘汰策略（LRU）**：TTL 只解决「过期」，不解决「条数太多」。进程内缓存要限制最大条数，满了就淘汰最久没用的。可以看标准库 `functools.lru_cache`，但注意它不能设 TTL，且对 async 函数不友好。
- **进程外缓存（Redis）**：多实例部署时，进程内字典各自为政，A 实例缓存的 B 实例看不到。要共享才上 Redis，并注意引入序列化和网络开销。
- **主动失效**：TTL 是「被动过期」，如果某条数据明确变了（比如产品改价），应该主动删掉对应 key，而不是等它自然过期。

---

## 第 6 节：常见坑与易错点

**坑 1：缓存了「易变 / 用户相关」的数据**

- 现象：A 用户看到了 B 用户的私密数据，或者价格显示成昨天的旧值。
- 原因：缓存 key 没有把「用户」「时效敏感」因素算进去，不同用户命中同一条缓存。
- 解决：凡是跟具体用户、实时状态相关的结果，**要么不缓存，要么把 userId 也放进 key**。缓存的默认前提是「相同输入一段时间内结果相同」，不满足这个前提就别缓存。

**坑 2：缓存键拼了「可变对象」，导致永不命中**

- 报错/现象：代码看起来对，但缓存永远 miss，一直在重复调上游。
- 原因：`dict` / `list` 这种可变对象当字典 key 会直接 `TypeError: unhashable type`；更隐蔽的是 key 拼了 `str(obj)` 而 obj 每次内存地址都变。
- 解决：缓存键要**归一化成稳定字符串**——排序字段、固定顺序、或像示例 3 那样用哈希压成定长串。

**坑 3：用 `time.time()` 判断过期，被系统改时间坑了**

- 现象：缓存莫名其妙「永不失效」或「瞬间全失效」。
- 原因：`time.time()` 是墙上时钟，NTP 校时 / 改时区 / 手动对表会让它倒退或跳变，`expires_at - time.time()` 就乱套。
- 解决：判断「过了多久」一律用 `time.monotonic()`（见第 3 节）。`time.time()` 只该用来记「现在几点」给人看。

**坑 4：缓存了「带副作用」的操作**

- 现象：发邮件、扣款、下单这种操作「只执行了一次」或「执行了不该执行的一次」。
- 原因：副作用操作重复执行会出事（扣两次款），缓存又可能让第二次请求拿到第一次的结果而不执行——两个方向都错。
- 解决：缓存只该套在**纯读取**上（查数据、算结果）。任何会改变外部状态的操作不要缓存。

**坑 5：无锁缓存导致「缓存击穿」**

- 现象：服务冷启动、或某条缓存刚好过期的那一刻，上游收到一波密集请求，甚至把自己打崩。
- 原因：大量并发请求同时 miss，同时去调上游，缓存没起到任何缓冲作用。
- 解决：加 per-key 锁 + 双重检查（见示例 3），让同一时刻只有一个协程去取数。

> **和直觉相反的一点**：你以为「加了缓存就一定省钱」，但如果没做并发控制，缓存过期的一瞬间所有请求**同时**穿透去打上游，反而可能比不加缓存更惨——因为原本可能是错峰的，现在被缓存「同步」到了同一时刻一起打。

---

## 第 7 节：验证你学会了（自测题）

1. **复述**：TTL 缓存是什么、解决什么问题？「命中」「未命中」「read-through」分别指什么？（答案在第 1、2、3 节）

2. **默写**：不看文档，写一个最小 `CacheEntry` 类，包含 `value`、`expires_at`、`is_valid`，并用 `time.monotonic()` 实现。（答案在第 4 节示例 1）

3. **组合应用**：给一个 `async def fetch_weather(city: str)` 加上 TTL 缓存，要求：① 相同 city 在 5 分钟内直接命中；② 不同 city 互不干扰；③ 并发下同一个 city 只调一次上游。（答案在第 4 节示例 3、第 5 节模板）


