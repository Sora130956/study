# Requests Quickstart

> **来源:** https://docs.python-requests.org/projects/cn/zh-cn/latest/user/quickstart.html
> **对应计划周次:** 第 2 周 · 周一 requests 文档精读 + HTTP 基础概念

## 核心理解

Requests 是 Python 里最常用的 HTTP 客户端库，把「发一个 HTTP 请求、拿到响应、解析内容」这套流程封装成极简 API。每种 HTTP 方法（GET/POST/PUT/DELETE/HEAD/OPTIONS）都有一个对应的同名函数，调用后返回一个 `Response` 对象，请求需要的所有信息（状态码、响应头、正文、cookie、重定向历史）都挂在这个对象上。

对接单场景，requests 是「调公开 API 取数据」和「爬虫抓网页」的入口：先用 requests 把数据拿回来（`r.json()` / `r.text`），再交给 BeautifulSoup 解析 HTML 或 pandas 处理结构化数据。掌握 `params`、`json`、`headers`、`timeout`、`raise_for_status()` 这几个点，基本能覆盖日常 API 调用的绝大多数需求。

## 关键点

### 发送请求

> `r = requests.get('https://api.github.com/events')`

每种 HTTP 方法对应一个函数，返回 `Response` 对象。POST 用 `data=` 传表单数据：

```python
import requests

r = requests.get('https://api.github.com/events')
r = requests.post('http://httpbin.org/post', data={'key': 'value'})
r = requests.put('http://httpbin.org/put', data={'key': 'value'})
r = requests.delete('http://httpbin.org/delete')
r = requests.head('http://httpbin.org/get')
r = requests.options('http://httpbin.org/get')
```

### 传递 URL 参数（params）

> Requests 允许你使用 `params` 关键字参数，以一个字符串字典来提供这些参数。

不用手工拼接 `?key=val`，把参数塞进字典交给 `params`，requests 自动编码到 URL 查询字符串。值为 `None` 的键不会加进去；值是列表时会重复出现同名键。

```python
payload = {'key1': 'value1', 'key2': ['value2', 'value3']}
r = requests.get('http://httpbin.org/get', params=payload)
print(r.url)  # http://httpbin.org/get?key1=value1&key2=value2&key2=value3
```

### 响应内容（text / content / json）

> Requests 会基于 HTTP 头部对响应的编码作出有根据的推测。当你访问 `r.text` 之时，Requests 会使用其推测的文本编码。

- `r.text`：文本内容，requests 按推测编码解码；可用 `r.encoding` 查看/修改编码，改了之后再访问 `r.text` 会用新编码重新解码。
- `r.content`：字节形式（`bytes`），用于非文本响应（如图片二进制）；自动处理 `gzip` / `deflate`。
- `r.json()`：内置 JSON 解码器，直接把响应体解析成 Python 对象。解码失败会抛异常。

```python
r = requests.get('https://api.github.com/events')
data = r.json()  # list / dict
```

⚠️ `r.json()` 调用成功 **不代表** 请求成功——有的服务器在失败响应里也返回 JSON（如 HTTP 500 的错误细节）。判断成功要看 `r.status_code` 或调 `r.raise_for_status()`。

### 定制请求头（headers）

> 如果你想为请求添加 HTTP 头部，只要简单地传递一个 `dict` 给 `headers` 参数就可以了。

```python
headers = {'user-agent': 'my-app/0.0.1'}
r = requests.get(url, headers=headers)
```

注意几个优先级规则：`.netrc` 里的认证信息会让 `headers=` 里设的授权失效（但 `auth=` 参数优先于 `.netrc`）；重定向到别的主机时授权头会被删除；能判断内容长度时 `Content-Length` 会被自动改写。所有 header 值必须是 string / bytestring / unicode。

### 更复杂的 POST（data / json / files）

> 你想要发送一些编码为表单形式的数据……只需简单地传递一个字典给 `data` 参数。

- `data=dict`：编码为表单形式（`application/x-www-form-urlencoded`）。
- `data=` 传元组列表：允许同一 key 多个值。
- `data=string`：直接原样发出（比如自己 `json.dumps` 后的字符串）。
- `json=dict`：requests 自动把字典编码成 JSON 并设好 header（2.4.2 新增，推荐调 JSON API 时用）。
- `files=`：上传多部分编码文件，强烈建议用二进制模式 `open(path, 'rb')`。

```python
# 调 JSON API（最常用）
r = requests.post(url, json={'some': 'data'})

# 上传文件
files = {'file': open('report.xls', 'rb')}
r = requests.post(url, files=files)
```

### 响应状态码与异常抛出

> 如果发送了一个错误请求(4XX / 5XX)，我们可以通过 `Response.raise_for_status()` 来抛出异常。

```python
r = requests.get('http://httpbin.org/get')
r.status_code                       # 200
r.status_code == requests.codes.ok  # True，内置状态码查询对象

r.raise_for_status()  # 2XX 返回 None；4XX/5XX 抛 HTTPError
```

调 API 的标准姿势：请求后先 `r.raise_for_status()`，再 `r.json()`，把网络/服务端错误挡在解析之前。

### 响应头（headers）

> HTTP 头部是大小写不敏感的。

`r.headers` 是一个大小写不敏感的字典，`r.headers['Content-Type']` 和 `r.headers.get('content-type')` 等价。服务器多次返回的同名 header 会被合并成用逗号分隔的一个值。

### Cookie

```python
r.cookies['example_cookie_name']              # 读取响应 cookie
r = requests.get(url, cookies={'a': 'b'})     # 发送 cookie
```

Cookie 返回对象是 `RequestsCookieJar`，行为类似字典但支持跨域名跨路径。

### 重定向与请求历史

> 默认情况下，除了 HEAD, Requests 会自动处理所有重定向。

- `r.url`：最终落地的 URL；`r.history`：为完成请求产生的中间 `Response` 列表（从老到新）。
- `allow_redirects=False` 可禁用重定向（对 GET/POST/PUT/PATCH/DELETE/OPTIONS 有效）。
- HEAD 默认不跟随，可用 `allow_redirects=True` 开启。

```python
r = requests.get('http://github.com')
r.url        # 'https://github.com/'
r.history    # [<Response [301]>]
```

### 超时（timeout）— 生产必写

> 基本上所有的生产代码都应该使用这一参数。如果不使用，你的程序可能会永远失去响应。

```python
requests.get('http://github.com', timeout=5)  # 单位：秒
```

⚠️ `timeout` 是「服务器多少秒内没响应就抛异常」（连接/读取的等待上限），**不是**整个下载响应的总时间限制。不写 timeout 请求可能永久挂起。

### 错误与异常

- `ConnectionError`：网络问题（DNS 失败、拒绝连接等）。
- `HTTPError`：`raise_for_status()` 遇到不成功状态码抛出。
- `Timeout`：请求超时。
- `TooManyRedirects`：超过最大重定向次数。
- 所有显式异常都继承自 `requests.exceptions.RequestException`（可用它做统一兜底捕获）。

```python
try:
    r = requests.get(url, timeout=5)
    r.raise_for_status()
    data = r.json()
except requests.exceptions.RequestException as e:
    print(f'请求失败: {e}')
```

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| Response | n. | requests 返回的响应对象，承载状态码/头/正文/cookie 等 |
| query string | n. | URL 中 `?` 后面的查询参数字符串 |
| params | n. | get 传 URL 查询参数的关键字参数 |
| payload | n. | 请求携带的数据（表单/JSON 等），此处指参数字典 |
| encoding | n. | 文本编码，requests 用 `r.encoding` 解码 `r.text` |
| status code | n. | HTTP 状态码，如 200、404、500 |
| raise_for_status | v. | 遇到 4XX/5XX 时主动抛 HTTPError 的方法 |
| redirect | n./v. | 重定向，服务器让客户端转向另一个 URL |
| history | n. | 重定向过程中产生的中间 Response 列表 |
| timeout | n. | 等待服务器响应的秒数上限，超时抛 Timeout |
| Multipart-Encoded | adj. | 多部分编码，用于表单上传文件的编码方式 |
| CookieJar | n. | 存储和管理 cookie 的容器对象 |
