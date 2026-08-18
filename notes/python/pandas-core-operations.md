# pandas 接单场景手册（AI 协作版）

> 对应计划：第二周（8月18日–22日）周四
> 定位：接单实战手册。记住 pandas **能做什么**，看懂每个场景的**代码大致结构**；
> 语法细节、参数细节不背，实际写代码交给 AI，自己负责 review 和自测。
> 配套笔记：零基础学习教程见 [pandas-tutorial.md](./pandas-tutorial.md)（先学那个，再看这个）。

---

## pandas 能力地图（记住这张表就够了）

| 大类 | 能力 | 对应函数 |
|------|------|---------|
| 读 | 读 CSV / Excel / JSON | `read_csv` / `read_excel` |
| 看 | 看前几行、行列数、类型、统计概览 | `head` / `shape` / `info` / `describe` |
| 选 | 选列、按条件选行 | `df['列']` / `df[条件]` |
| 改 | 加列、改类型、去重、填/删空值 | 赋值 / `astype` / `drop_duplicates` / `fillna` / `dropna` |
| 算 | 分组统计、透视表 | `groupby().agg()` / `pivot_table` |
| 合 | 纵向拼接、横向关联（类似 SQL JOIN） | `concat` / `merge` |
| 写 | 写 CSV / Excel（可多 sheet） | `to_csv` / `to_excel` / `ExcelWriter` |

**核心数据结构就两个：**
- `DataFrame` = 一张 Excel 表（行 + 列）
- `Series` = 表里的一列

**两个最常见的坑（记住，省调试时间）：**
1. 多条件筛选用 `&` / `|` 且每个条件加括号，不是 `and` / `or`
2. 中文 CSV 读写乱码：读写都加 `encoding='utf-8-sig'`

---

## 场景 1：拿到文件，先看看数据长什么样

**接单背景**：客户丢过来一个 CSV，第一步永远是先摸清结构。

```python
import pandas as pd

df = pd.read_csv('orders.csv', encoding='utf-8-sig')

print(df.head())        # 前5行长什么样
print(df.shape)         # (行数, 列数)
print(df.dtypes)        # 每列是什么类型（数字？文本？日期？）
print(df.isna().sum())  # 每列有几个空值（判断数据脏不脏）
```

**review 要点**：日期列经常被读成文本（`object`），金额列可能混着货币符号——后面清洗就是处理这些。

---

## 场景 2：筛选 + 清洗脏数据

**接单背景**："帮我把无效数据去掉，只留符合xxx条件的"。

```python
# 筛选：单条件
df = df[df['金额'] > 0]

# 筛选：多条件（注意 & 和括号）
df = df[(df['金额'] > 0) & (df['状态'] == '已完成')]

# 清洗：去重
df = df.drop_duplicates()

# 清洗：删掉关键列为空的行
df = df.dropna(subset=['金额'])

# 清洗：空值填默认值
df['备注'] = df['备注'].fillna('无')

# 类型转换：文本日期 → 真日期，文本数字 → 真数字
df['日期'] = pd.to_datetime(df['日期'])
df['金额'] = pd.to_numeric(df['金额'], errors='coerce')  # 转不了的变 NaN
```

**review 要点**：`errors='coerce'` 会把脏数据变成 NaN，转换后记得再 `dropna` 一次。

### 日期转换：不传 format 为什么能转？

`pd.to_datetime` 有**自动推断**机制，常见格式都认识：

```python
pd.to_datetime('2026-08-13')        # ISO 格式 ✓
pd.to_datetime('2026/08/13')        # 斜杠分隔 ✓
pd.to_datetime('2026-08-13 14:30')  # 带时间 ✓
```

**但有歧义时会猜错**：

```python
pd.to_datetime('01/02/2026')
# 是 1月2日 还是 2月1日？pandas 默认按美式（月/日/年）解析
```

**稳妥做法——显式传 format 或 dayfirst**：

```python
df['日期'] = pd.to_datetime(df['日期'], format='%Y-%m-%d')   # 2026-08-13
df['日期'] = pd.to_datetime(df['日期'], format='%d/%m/%Y')   # 13/08/2026
df['日期'] = pd.to_datetime(df['日期'], dayfirst=True)       # 明确"日在前"
```

**经验法则**：
- 标准 ISO 格式（`YYYY-MM-DD`）→ 不传，pandas 有优化，又快又准
- 格式统一但非 ISO → 传 `format`
- 格式混乱 → 先让 AI 写清洗规则统一格式，再转

### 金额转换：float 精度问题

`pd.to_numeric` 默认返回 `float64`，**float 有二进制精度问题**：

```python
0.1 + 0.2   # 0.30000000000000004
```

**影响评估**：

| 场景 | 风险 |
|------|------|
| 日常报表（保留2位小数） | 几乎无感，float64 能精确表示 15~16 位有效数字 |
| 财务对账（分毫不差） | 有风险，应该用整数（分）或 Decimal |
| 超大金额（万亿级）+ 小数 | 有风险 |

**应对方案**：

```python
# 方案1：转成分（整数），避免小数
df['金额_分'] = (df['金额'] * 100).round().astype('Int64')

# 方案2：用 Decimal 精确类型
from decimal import Decimal
df['金额'] = df['金额'].apply(Decimal)

# 方案3：保持 float，展示时格式化
df['金额'] = pd.to_numeric(df['金额'])
df['金额显示'] = df['金额'].apply(lambda x: f'{x:.2f}')
```

**接单建议**：客户没提精度要求 → 方案3 够用；财务/会计客户 → 主动确认精度要求，用方案1或2。

---

## 场景 3：分组统计出报表（最高频需求）

**接单背景**："按地区/按月/按品类给我统计一下"。这是 Upwork 上最常见的数据处理单子。

```python
# 按城市分组，算多种统计
result = df.groupby('城市').agg(
    总金额=('金额', 'sum'),
    平均金额=('金额', 'mean'),
    订单数=('金额', 'count')
).reset_index()

print(result)
```

输出：

```
   城市      总金额     平均金额  订单数
0  北京  150000.00  7500.00   20
1  上海   98000.00  8166.67   12
```

**代码结构就三步**：`groupby(按什么分)` → `agg(新列名=(对哪列, 算什么))` → `reset_index()`。

**常用统计**：`sum` 总和 / `mean` 平均 / `max` / `min` / `count` 数量 / `nunique` 去重计数。

**按月统计**（从日期列提取月份再分组）：

```python
df['月份'] = df['日期'].dt.to_period('M')   # 2026-08-13 → 2026-08
月报 = df.groupby('月份').agg(总金额=('金额', 'sum')).reset_index()
```

---

## 场景 4：合并多个文件

**接单背景**："我每个月一个 Excel，帮我合并成一个"——经典单子。

```python
import glob

# 找到所有文件
files = glob.glob('data/*.xlsx')        # data 文件夹下所有 xlsx

# 逐个读入，纵向拼接（行追加）
df = pd.concat([pd.read_excel(f) for f in files], ignore_index=True)

df.to_excel('合并结果.xlsx', index=False)
```

**review 要点**：各文件列名必须一致才能拼；如果不一致，先逐个 `rename` 对齐列名再 `concat`。

---

## 场景 5：多表关联（类似 SQL JOIN / Excel VLOOKUP）

**接单背景**："订单表里有商品ID，商品表里有商品名称和价格，帮我合到一张表"。

```python
orders = pd.read_excel('订单.xlsx')      # 有 商品ID、数量
products = pd.read_excel('商品.xlsx')    # 有 商品ID、名称、单价

# 按 商品ID 把两张表拼起来（左连接：保留所有订单）
df = pd.merge(orders, products, on='商品ID', how='left')

# 拼完直接算金额
df['金额'] = df['数量'] * df['单价']
```

**`how` 参数对应 SQL JOIN：**

| 写法 | 效果 |
|------|------|
| `how='left'` | 保留左表全部行（最常用，防止订单丢数据） |
| `how='inner'` | 只保留两边都有的 |
| `how='outer'` | 两边全保留，缺的填 NaN |

**review 要点**：关联后行数变多了？说明关联键在右表里有重复，先 `products.drop_duplicates(subset=['商品ID'])`。

### 透视表补充：什么是透视表？和 groupby 的区别？

**透视表 = 把"行"和"列"交叉，中间填统计值。** Excel 里叫"数据透视表"，pandas 里就是 `pivot_table`。

**能看出聚合的例子**（同一个"月份+品类"有多条记录）：

```python
df = pd.DataFrame({
    '月份': ['2026-08', '2026-08', '2026-08', '2026-09', '2026-09', '2026-09'],
    '品类': ['手机', '手机', '电脑', '手机', '手机', '电脑'],
    '金额': [100, 200, 300, 150, 250, 400]
})
```

数据：

```
     月份  品类  金额
0  2026-08  手机  100
1  2026-08  手机  200   ← 同一个月同一个品类，两条数据
2  2026-08  电脑  300
3  2026-09  手机  150
4  2026-09  手机  250   ← 同一个月同一个品类，两条数据
5  2026-09  电脑  400
```

**`groupby` 结果（长表）**：

```python
df.groupby(['月份', '品类'])['金额'].sum()
# 月份      品类
# 2026-08  电脑    300
#          手机    300   ← 100 + 200 = 300，聚合了！
# 2026-09  电脑    400
#          手机    400   ← 150 + 250 = 400，聚合了！
```

**`pivot_table` 结果（宽表，品类展开成列）**：

```python
pd.pivot_table(df, values='金额', index='月份', columns='品类', aggfunc='sum')
# 品类      电脑   手机
# 月份
# 2026-08  300   300
# 2026-09  400   400
```

**`pivot_table` 参数对应 Excel 透视表**：

| 参数 | 作用 | Excel 透视表位置 |
|------|------|----------------|
| `index='月份'` | 行标签 | 行 |
| `columns='品类'` | 列标签 | 列 |
| `values='金额'` | 要统计的值 | 值 |
| `aggfunc='sum'` | 怎么算（求和/平均/计数） | 值字段设置 |
| `fill_value=0` | 空格子填 0 | 空单元格显示 |

**什么时候用哪个？**
- 要"长表"继续处理 → `groupby`
- 要"宽表"直接看/导出报表 → `pivot_table`

### 缺失值补充：`na` 是什么？`dropna` 删的是什么？

**`na` = Not Available**（不可用），表示缺失值。相关缩写：

| 缩写 | 全称 | 含义 |
|------|------|------|
| `NA` | Not Available | 缺失值统称 |
| `NaN` | Not a Number | 数值计算中的非法/缺失结果（来自 NumPy） |
| `NaT` | Not a Time | 日期时间类型的缺失值 |

**`dropna()` 删的是所有"真缺失"**，包括：

| 值 | 类型 | `isna()` 结果 |
|----|------|-------------|
| `None` | Python 原生空 | True |
| `np.nan` | NumPy 浮点缺失 | True |
| `pd.NA` | pandas 新类型缺失 | True |
| `pd.NaT` | 日期缺失 | True |
| `''`（空字符串） | 文本 | **False**（空字符串不是缺失值！） |

**坑：空字符串 `''` 不会被 `dropna` 删掉！**

```python
df['备注'].fillna('')        # 把缺失填成空字符串
df.dropna(subset=['备注'])   # 空字符串还在表里！

# 正确做法：先把空字符串转成 NA，再删
df['备注'] = df['备注'].replace('', pd.NA)
df = df.dropna(subset=['备注'])
```

---

## 场景 6：一个表拆成多个文件

**接单背景**：和场景 4 相反——"按部门把这个总表拆成单独的 Excel 发给我"。

```python
df = pd.read_excel('总表.xlsx')

# 按部门分组，每组写出一个文件
for 部门, 子表 in df.groupby('部门'):
    子表.to_excel(f'输出/{部门}.xlsx', index=False)
```

**代码结构**：`groupby` 遍历 → 每次循环拿到 `(分组值, 该组的 DataFrame)` → 分别导出。

---

## 场景 7：完整实战——脏 CSV 进，多 sheet Excel 报告出

**接单背景**：把前面所有能力串起来，这就是一个完整可交付的"数据处理脚本"单子。

**需求**：客户的销售 CSV → 清洗 → 月度汇总 + 品类透视 → 输出格式化 Excel。

```python
import pandas as pd

# 1. 读取 + 总览
df = pd.read_csv('sales.csv', encoding='utf-8-sig')
print(df.head())
print(df.isna().sum())

# 2. 清洗
df = df.dropna(subset=['金额', '日期'])
df = df.drop_duplicates()
df['金额'] = pd.to_numeric(df['金额'], errors='coerce')
df = df.dropna(subset=['金额'])
df = df[df['金额'] > 0]
df['日期'] = pd.to_datetime(df['日期'])
df['月份'] = df['日期'].dt.to_period('M')

# 3. 统计：月度汇总
月报 = df.groupby('月份').agg(
    总金额=('金额', 'sum'),
    平均单笔=('金额', 'mean'),
    订单数=('金额', 'count')
).reset_index()

# 4. 统计：品类 × 月份透视表
透视 = pd.pivot_table(
    df, values='金额', index='月份', columns='品类',
    aggfunc='sum', fill_value=0
)

# 5. 导出：一个 Excel 三个 sheet
with pd.ExcelWriter('销售报告.xlsx') as writer:
    月报.to_excel(writer, sheet_name='月度汇总', index=False)
    透视.to_excel(writer, sheet_name='品类透视')
    df.to_excel(writer, sheet_name='清洗后明细', index=False)

print('完成：销售报告.xlsx')
```

**这就是交付物本身。** 客户每月把新 CSV 放进文件夹，跑一遍脚本就出报告。

---

## 附录 A：速查表（review AI 代码时对照用）

| 我想… | 代码 |
|-------|------|
| 读 CSV | `pd.read_csv('f.csv', encoding='utf-8-sig')` |
| 读 Excel | `pd.read_excel('f.xlsx')` |
| 看前几行 | `df.head()` |
| 看行列数 | `df.shape` |
| 选一列 | `df['列名']` |
| 筛选行 | `df[df['列名'] == '值']` |
| 多条件筛选 | `df[(条件1) & (条件2)]` |
| 排序 | `df.sort_values('列名', ascending=False)` |
| 加新列 | `df['新列'] = df['旧列'] * 2` |
| 删空值行 | `df.dropna(subset=['关键列'])` |
| 空值填充 | `df['列'].fillna(默认值)` |
| 去重 | `df.drop_duplicates()` |
| 日期转换 | `pd.to_datetime(df['日期'])` |
| 数字转换 | `pd.to_numeric(df['列'], errors='coerce')` |
| 分组统计 | `df.groupby('列').agg(新名=('列', 'sum'))` |
| 纵向合并 | `pd.concat([df1, df2], ignore_index=True)` |
| 横向关联 | `pd.merge(df1, df2, on='键', how='left')` |
| 透视表 | `pd.pivot_table(df, values=, index=, columns=, aggfunc=)` |
| 导出 CSV | `df.to_csv('f.csv', index=False, encoding='utf-8-sig')` |
| 导出 Excel | `df.to_excel('f.xlsx', index=False)` |
| 多 sheet | `with pd.ExcelWriter('f.xlsx') as w:` |

---

## 附录 B：遇到问题怎么问 AI

**好的提问 = 输入什么样 + 想要什么样 + 现在卡在哪：**

> "我有一个 CSV，列是：日期（2026-08-13 格式文本）、金额（有的带¥符号）、城市。
> 我想要：按月统计总金额，输出 Excel。
> 现在 `pd.to_datetime` 报错 / 金额求和结果是文本拼接，怎么修？"

比"pandas 怎么按月统计"得到的答案质量高得多。

---

## 附录 C：原理性备注（不用记，review 时备查）

以下内容是学习过程中答疑记录的，**属于语法糖细节，不需要背**。实际写代码交给 AI 时这些不重要，但 review 时遇到可以回来对照。

### `df['城市']` 为什么返回的不是列表？

`df['城市']` 看起来像 dict 取值，但返回的是 `Series` 对象（带索引、列名、dtype）。

**原因**：`[]` 在 Python 里不是 dict 专用语法，任何类实现 `__getitem__` 方法就能用 `[]`：

- `dict.__getitem__` → 返回 value
- `list.__getitem__` → 返回第 n 个元素
- `DataFrame.__getitem__` → 返回一列（Series）

（Java 不允许重载操作符，所以 Java 里必须显式调方法如 `df.getColumn("城市")`。）

```python
col = df['城市']
print(type(col))        # <class 'pandas.core.series.Series'>
print(col.tolist())     # ['北京', '上海'] ← 转回普通 list
```

`[]` 在 DataFrame 上按传入类型不同行为不同：

```python
df['城市']              # 传字符串     → 一列（Series）
df[['城市', '工资']]    # 传列表       → 多列（DataFrame）
df[df['工资'] > 10000]  # 传布尔Series → 筛选后的行（DataFrame）
```

### `df['城市'] == '北京'` 为什么能直接筛选？

`==` 也被重载了：`Series.__eq__('北京')` 是**对每个元素逐个比较**，返回布尔 Series `[True, False, True, ...]`。这种"一个操作作用于所有元素"叫**向量化（vectorization）**。

所以筛选是两步重载的组合：

```python
df[df['城市'] == '北京']
#      ↑________________↑
#      ① Series.__eq__ → 布尔 Series
#   ↑__________________________↑
#   ② DataFrame.__getitem__(布尔Series) → 保留 True 的行
```

同理，`df['工资'] * 12`、`df['工资'] > 10000` 都是向量化操作。

### `agg(平均工资=('工资', 'mean'))` 是什么语法？

这是 Python 的**关键字参数**，不是字典：

- `平均工资` 是参数名（Python 3 允许中文标识符），不是字符串，不用引号
- `('工资', 'mean')` 是元组：`(源列名, 聚合函数)`
- `agg` 内部用 `**kwargs` 接收：`{'平均工资': ('工资', 'mean'), ...}`，参数名变成新列名

`**kwargs` 核心理解：`func(姓名='张三')` 等价于 `func(**{'姓名': '张三'})`——参数名变成 dict 的键，参数值变成 dict 的值。适合"参数名本身就是数据"的场景（比如聚合时自定义新列名）。

参数名有空格/特殊字符时，只能退化到字典写法：`agg(**{'平均 工资': ('工资', 'mean')})`。
