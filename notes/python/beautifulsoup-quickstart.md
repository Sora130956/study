# BeautifulSoup Quickstart

> **来源:** https://beautifulsoup.readthedocs.io/zh-cn/v4.4.0/ ，https://www.crummy.com/software/BeautifulSoup/bs4/doc/
> **对应计划周次:** 第 2 周 · 周二 BeautifulSoup 文档精读 + 爬豆瓣 Top250

## 核心理解

BeautifulSoup（简称 BS4）是把一坨 HTML/XML 字符串**解析成一棵可查询的树**、再从中精准提取数据的库。它和 requests 是爬虫的标准搭档：requests 负责把网页抓回来（得到 `response.text` 这坨 HTML 字符串），BS4 负责从这坨字符串里定位并抠出你要的内容（标题、评分、链接等）。

用 Java 类比：这类似 Jsoup。你把 HTML 交给 `BeautifulSoup(html, "html.parser")`，得到一个 `soup` 对象（整棵树的根），然后用 `find` / `find_all` / `select` 像查询一样定位节点，用 `.text` 取文本、用 `["属性名"]` 取标签属性。核心就这么几个方法，覆盖 90% 的爬虫解析需求。

导入写法：`from bs4 import BeautifulSoup`（包名装的是 `beautifulsoup4`，但导入用 `bs4`）。

## 关键点

### 创建 soup 对象（解析入口）

> `BeautifulSoup(markup, parser)` 把 HTML 字符串解析成文档树。

```python
import requests
from bs4 import BeautifulSoup

html = requests.get(url, headers=headers, timeout=5).text
soup = BeautifulSoup(html, "html.parser")
```

第二个参数是解析器，先用 Python 内置的 `"html.parser"`（不用装额外库）。`"lxml"` 更快但要 `pip install lxml`，练习阶段用内置的即可。

### 四种核心对象

BS4 把树里的东西分成几类，最常打交道的两个：

- **Tag**：一个标签节点，如 `<span class="title">肖申克</span>`。有 `.name`（标签名）、属性、子节点。
- **NavigableString**：标签里的文本内容，一般不直接用，靠 `.text` 拿。

### find / find_all —— 最核心的定位方法

> `find()` 返回第一个匹配的 Tag；`find_all()` 返回所有匹配的**列表**。

```python
# 按标签名
soup.find("span")            # 第一个 <span>
soup.find_all("div")         # 所有 <div>，返回 list

# 按 class（注意是 class_ ，带下划线，因为 class 是 Python 关键字）
soup.find("span", class_="title")
soup.find_all("div", class_="item")

# 按 id
soup.find("div", id="content")

# 组合多个属性
soup.find("a", attrs={"data-id": "123"})
```

⚠️ 关键坑：`class` 是 Python 保留字，所以按 class 查要写成 **`class_`**（带下划线）。

`find_all` 返回列表，要遍历：

```python
items = soup.find_all("div", class_="item")
for item in items:
    print(item.find("span", class_="title").text)
```

### select / select_one —— 用 CSS 选择器定位

> `select()` 用 CSS 选择器返回**列表**；`select_one()` 返回第一个。

如果你熟 CSS 选择器（前端背景或写过 jQuery），这套比 find 更顺手：

```python
soup.select_one("span.title")        # class 为 title 的第一个 span
soup.select("div.item")              # 所有 class 为 item 的 div
soup.select("#content")              # id 为 content 的元素
soup.select("div.item span.title")   # 后代选择器：item 里的 title
soup.select("ol.grid_view li")       # 豆瓣 Top250 常见结构
```

CSS 选择器速查：
- `tag` 标签名 / `.class` 类名 / `#id` 主键
- `a b` 后代（a 里所有 b）/ `a > b` 直接子级
- `[attr=value]` 按属性

find 和 select 二选一即可，效果一样。爬豆瓣两种都能用。

### 取文本：.text / .get_text() / .string

```python
tag = soup.find("span", class_="title")
tag.text            # 标签内所有文本（含子标签文本），最常用
tag.get_text()      # 同 .text，可传参 get_text(strip=True) 去空白
tag.string          # 仅当标签只有一个纯文本子节点时才有值，否则 None
```

实战建议统一用 `.text`，再配 `.strip()` 去掉首尾空白换行：

```python
title = tag.text.strip()
```

### 取标签属性：像 dict 一样用中括号

> 访问标签的属性用 `tag["属性名"]`。

```python
a = soup.find("a")
a["href"]              # 取 href 属性值
a["class"]             # 注意：class 返回的是 list（可能多个类）
a.get("href")          # 更安全，属性不存在返回 None 而非报错
img = soup.find("img")
img["src"]
```

呼应之前那条笔记：**方法调用用点号（`find`/`get`），取标签属性用中括号（`tag["href"]`）**，别混。

### 爬豆瓣的完整骨架（把上面串起来）

```python
import requests
from bs4 import BeautifulSoup

headers = {"User-Agent": "Mozilla/5.0"}
html = requests.get("https://movie.douban.com/top250", headers=headers, timeout=5).text
soup = BeautifulSoup(html, "html.parser")

for item in soup.find_all("div", class_="item"):
    title = item.find("span", class_="title").text.strip()
    rating = item.find("span", class_="rating_num").text.strip()
    print(title, rating)
```

### 反爬提醒：必须带 User-Agent

豆瓣直接 `requests.get` 不带 header 会返回 418/403（被识别为爬虫）。加上 `headers={"User-Agent": "..."}` 伪装成浏览器一般就能过——这正是 requests 笔记「定制请求头」那节的实际应用。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| parser | n. | 解析器，把 HTML 字符串解析成树，如 `html.parser` |
| Tag | n. | 标签节点对象，对应一个 HTML 标签 |
| NavigableString | n. | 标签内的文本内容对象 |
| find | v. | 返回第一个匹配的标签 |
| find_all | v. | 返回所有匹配标签的列表 |
| select | v. | 用 CSS 选择器返回匹配标签列表 |
| CSS selector | n. | CSS 选择器，如 `div.item span.title` |
| attribute | n. | 标签属性，如 href、class、src |
| User-Agent | n. | 请求头字段，标识客户端，反爬绕过常用 |
