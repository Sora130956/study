# pandas 入门教程：从零开始

> 对应计划：第二周（8月18日–22日）周四
> 适合：完全没接触过 pandas，但会 Python 基础语法的人
> 学习方式：边看边敲，每个代码块都可以直接运行

---

## 第 0 章：pandas 是干啥的？

**一句话：pandas 就是 Python 版的 Excel。**

你在 Excel 里干的活——打开表格、筛选行、按列排序、分类汇总、做透视表——用 pandas 几行代码就能搞定。区别是：

| Excel | pandas |
|-------|--------|
| 鼠标点来点去 | 写代码，一次写好反复用 |
| 几十万行就卡死 | 几百万行也扛得住 |
| 手动操作无法复现 | 脚本可以重复跑、可以自动化 |
| 合并10个表要复制粘贴半天 | 一个循环搞定 |

**为什么接单要用它？** Upwork 上大量 Python 单子就是这类需求：
- "我有 50 个 Excel 报表，帮我合并成一个"
- "帮我从这个 CSV 里筛选出符合某些条件的数据，输出成 Excel"
- "帮我每月自动生成销售统计报表"

这些活手动干要几小时，写个 pandas 脚本几秒钟跑完——这就是你的价值。

**安装：**

```bash
pip install pandas openpyxl
```

（openpyxl 是让 pandas 能读写 Excel 文件的依赖库）

---

## 第 1 章：核心概念——一张表

pandas 里最重要的东西叫 **DataFrame**，你就把它理解成**一张 Excel 表**：

```python
import pandas as pd

df = pd.DataFrame({
    '姓名': ['张三', '李四', '王五', '赵六'],
    '年龄': [25, 30, 35, 28],
    '城市': ['北京', '上海', '北京', '广州'],
    '工资': [8000, 12000, 15000, 9000]
})

print(df)
```

输出：

```
   姓名  年龄  城市     工资
0  张三  25  北京   8000
1  李四  30  上海  12000
2  王五  35  北京  15000
3  赵六  28  广州   9000
```

看到没，就是一张表：
- 有**列名**（姓名、年龄、城市、工资）
- 左边那一列 0、1、2、3 叫**索引（index）**，相当于 Excel 的行号
- 整个表就是 `df`（DataFrame 的变量名习惯叫 df）

**一列**单独拿出来叫 **Series**，就是一维的数据：

```python
print(df['姓名'])      # 取出"姓名"这一列
```

> 约定：全文开头都有 `import pandas as pd`，后面不再重复写。

---

## 第 2 章：读取文件——把 CSV 变成表

实际工作中数据都在文件里，最常见的是 **CSV 文件**（用记事本都能打开的那种，每行一条数据，逗号分隔）。

假设有个 `employees.csv` 文件，内容是：

```csv
姓名,年龄,城市,工资
张三,25,北京,8000
李四,30,上海,12000
王五,35,北京,15000
赵六,28,广州,9000
孙七,42,北京,18000
```

读取只要一行：

```python
df = pd.read_csv('employees.csv')
print(df)
```

**读 Excel 文件也一样简单：**

```python
df = pd.read_excel('employees.xlsx')
```

**常见坑——中文乱码：**

```python
# 如果报编码错误或读出来是乱码，试试指定编码
df = pd.read_csv('employees.csv', encoding='utf-8')
df = pd.read_csv('employees.csv', encoding='gbk')   # Windows 中文系统常见
```

---

## 第 3 章：先认识你的数据——查看基本信息

拿到一个陌生的表，第一件事永远是**先看看它长什么样**：

```python
df.head()        # 看前5行（最常用，防止表太大刷屏）
df.head(3)       # 看前3行
df.shape         # (5, 4) ← 5行4列
df.columns       # 有哪些列
df.info()        # 每列的类型、有没有空值
df.describe()    # 数值列的统计：平均值、最大最小值等
```

输出 `df.describe()` 示例：

```
             年龄           工资
count   5.000000      5.000000   ← 有5个值
mean   32.000000  12400.000000   ← 平均年龄32，平均工资12400
std     6.557439   4012.480598
min    25.000000   8000.000000   ← 最小值
max    42.000000  18000.000000   ← 最大值
```

这几行代码没有记忆负担，**每次拿到新数据固定跑一遍**就行。

---

## 第 4 章：筛选——"我要北京的人"

这是最核心的操作，对应 Excel 的"筛选"按钮。

**思路：写一个条件，得到你想要的行。**

```python
# 筛选：城市是北京的
df[df['城市'] == '北京']
```

结果：

```
   姓名  年龄  城市     工资
0  张三  25  北京   8000
2  王五  35  北京  15000
4  孙七  42  北京  18000
```

**这行代码怎么理解？** 拆开看：

```python
df['城市'] == '北京'
# 结果是一列 True/False：
# 0     True   ← 张三是北京的
# 1    False
# 2     True
# 3    False
# 4     True

df[一列True/False]   # 只保留 True 的行
```

**更多条件写法：**

```python
df[df['工资'] > 10000]                          # 工资大于1万
df[df['年龄'] <= 30]                            # 年龄小于等于30
df[df['城市'].isin(['北京', '广州'])]           # 城市是北京或广州

# 多个条件同时满足：用 & 连接，每个条件加括号
df[(df['城市'] == '北京') & (df['工资'] > 10000)]

# 满足任意一个：用 |
df[(df['年龄'] < 26) | (df['工资'] > 15000)]
```

> 注意：多个条件时必须用 `&` 和 `|`（不是 `and` / `or`），且每个条件都要用括号包起来。这是 pandas 最常见的报错来源。

**筛选后只想要某几列：**

```python
df.loc[df['城市'] == '北京', ['姓名', '工资']]
#        条件（行）          想要的列
```

---

## 第 5 章：排序

```python
df.sort_values('工资')                       # 按工资升序（从小到大）
df.sort_values('工资', ascending=False)      # 降序（从大到小）
df.sort_values(['城市', '工资'])             # 先按城市，再按工资
```

---

## 第 6 章：加列、改列

```python
# 直接赋值就能加新列
df['年薪'] = df['工资'] * 12

# 基于条件加列：给工资过万的人标"高薪"
df['级别'] = df['工资'].apply(lambda x: '高薪' if x > 10000 else '普通')

# 修改列名
df = df.rename(columns={'工资': '月薪'})

# 删除一列
df = df.drop(columns=['级别'])
```

`apply(lambda x: ...)` 的意思：**对列里每个值 x 执行一遍这个函数**。不懂 lambda 的话可以写普通函数：

```python
def 定级(x):
    if x > 10000:
        return '高薪'
    else:
        return '普通'

df['级别'] = df['工资'].apply(定级)
```

---

## 第 7 章：分组统计——本教程最重要的部分

**需求：每个城市的平均工资是多少？**

Excel 里你要做透视表，pandas 里一行：

```python
df.groupby('城市')['工资'].mean()
```

结果：

```
城市
上海    12000
北京    14333
广州     9000
Name: 工资, dtype: float64
```

**怎么理解：** `groupby('城市')` = 把数据按城市分成三堆（北京3人、上海1人、广州1人），然后对每堆的"工资"算平均值。

**常用统计函数：**

```python
df.groupby('城市')['工资'].mean()    # 平均值
df.groupby('城市')['工资'].sum()     # 总和
df.groupby('城市')['工资'].max()     # 最大
df.groupby('城市')['工资'].count()   # 数量
```

**一次算好几种统计（推荐写法）：**

```python
result = df.groupby('城市').agg(
    平均工资=('工资', 'mean'),
    最高工资=('工资', 'max'),
    人数=('姓名', 'count')
).reset_index()          # 把城市变回普通列，方便后续处理/导出

print(result)
```

结果：

```
   城市      平均工资   最高工资  人数
0  上海  12000.00  12000   1
1  北京  14333.33  18000   3
2  广州   9000.00   9000   1
```

**这个结果就是一张新的 DataFrame**，可以直接导出成 Excel 交给客户——这就完成了一个最简单的"数据处理"单子。

---

## 第 8 章：处理脏数据（真实世界必有）

真实数据经常是脏的：有空值、有重复。假设数据变成这样：

```
   姓名  年龄  城市     工资
0  张三  25  北京   8000
1  李四  30  上海  12000
2  王五  35  北京    NaN    ← 工资缺失
3  张三  25  北京   8000    ← 和第一行重复
```

```python
# 先看看哪里有空值
df.isna().sum()          # 每列有几个空值

# 处理空值：填一个值
df['工资'] = df['工资'].fillna(0)         # 空工资填0

# 或者：直接删掉有空值的行
df = df.dropna()

# 去重
df = df.drop_duplicates()
```

---

## 第 9 章：导出——把结果交给客户

```python
# 导出 CSV（encoding='utf-8-sig' 是为了 Excel 打开不乱码，记住这个固定搭配）
df.to_csv('result.csv', index=False, encoding='utf-8-sig')

# 导出 Excel
df.to_excel('result.xlsx', index=False)
```

> `index=False` 的意思是不要导出左边那列 0,1,2,3 的行号——客户不需要它。

**导出多个 sheet：**

```python
with pd.ExcelWriter('报告.xlsx') as writer:
    df.to_excel(writer, sheet_name='明细数据', index=False)
    result.to_excel(writer, sheet_name='城市统计', index=False)
```

---

## 第 10 章：完整实战——把前面所有东西串起来

**场景（模拟真实接单）：** 客户给你一个销售 CSV，要求：清洗数据 → 按月统计 → 输出 Excel 报告。

```python
import pandas as pd

# 1. 读取
df = pd.read_csv('sales.csv', encoding='utf-8-sig')

# 2. 先看看数据
print(df.head())
print(df.isna().sum())

# 3. 清洗：删掉金额为空的行，去重
df = df.dropna(subset=['金额'])
df = df.drop_duplicates()

# 4. 日期列转成真正的日期类型，并提取出"月份"
df['日期'] = pd.to_datetime(df['日期'])
df['月份'] = df['日期'].dt.to_period('M')    # 2026-08-13 → 2026-08

# 5. 只保留有效订单（金额大于0）
df = df[df['金额'] > 0]

# 6. 按月份分组统计
月报 = df.groupby('月份').agg(
    总金额=('金额', 'sum'),
    平均单笔=('金额', 'mean'),
    订单数=('金额', 'count')
).reset_index()

# 7. 导出 Excel 报告（两个 sheet：汇总 + 明细）
with pd.ExcelWriter('销售月报.xlsx') as writer:
    月报.to_excel(writer, sheet_name='月度汇总', index=False)
    df.to_excel(writer, sheet_name='订单明细', index=False)

print('搞定，已生成 销售月报.xlsx')
```

**这个脚本就是一个最小可用的"数据处理产品"。** 客户以后每月把新 CSV 丢给你（或者丢进文件夹），跑一遍就出报告。

---

## 第 11 章：学习路径建议

按你今天的计划（pandas 核心操作），建议按这个顺序练：

1. **先跑通本文所有代码**（约1小时）——用第1章那个小表，每个代码块都敲一遍、改改参数看看结果变化
2. **做一个练习**：自己造一个 CSV（或用你爬虫项目爬到的 books.csv），完成"读入 → 筛选 → 分组统计 → 导出 Excel"全流程
3. **遇到具体问题再查**：比如"怎么合并两个表"（`pd.merge`）、"怎么拆列"（`str.split`），用到再学，不用提前背

**今天不需要掌握的**（用到再说）：
- 透视表 `pivot_table`
- 多表合并 `merge` / `concat`
- 时间序列重采样 `resample`
- 多层索引 MultiIndex

---

## 附录：速查表

| 我想… | 代码 |
|-------|------|
| 读 CSV | `pd.read_csv('f.csv', encoding='utf-8-sig')` |
| 读 Excel | `pd.read_excel('f.xlsx')` |
| 看前几行 | `df.head()` |
| 看行数列数 | `df.shape` |
| 选一列 | `df['列名']` |
| 筛选行 | `df[df['列名'] == '值']` |
| 多条件筛选 | `df[(条件1) & (条件2)]` |
| 排序 | `df.sort_values('列名', ascending=False)` |
| 加新列 | `df['新列'] = df['旧列'] * 2` |
| 删空值行 | `df.dropna()` |
| 去重 | `df.drop_duplicates()` |
| 分组统计 | `df.groupby('列').agg(新名=('列', 'sum'))` |
| 导出 CSV | `df.to_csv('f.csv', index=False, encoding='utf-8-sig')` |
| 导出 Excel | `df.to_excel('f.xlsx', index=False)` |
