# openpyxl 快速认知手册

> 定位：**不用深入学，知道能干什么、什么时候用、代码让 AI 写**。
> 本文目标：你看一遍，知道 openpyxl 在接单流程里扮演什么角色。

---

## openpyxl 是干啥的？

**专门操作 Excel 文件（.xlsx）的 Python 库。**

和 pandas 的分工：

| 场景                   | 用什么                       |
| -------------------- | ------------------------- |
| 读 Excel 做数据分析        | pandas                    |
| 数据写进 Excel（只要数据对）    | pandas                    |
| 数据写进 Excel（还要调格式）    | pandas 写数据 + openpyxl 调格式 |
| 已有 Excel 模板，往指定单元格填数 | **openpyxl**（pandas 干不了）  |
| Excel 里写公式、插图表、条件格式  | **openpyxl**（pandas 干不了）  |

**一句话**：pandas 管数据，openpyxl 管格式 + 模板操作。

---

## openpyxl 能做什么？（知道这些就行）

### 1. 读写单元格（基础）

```python
from openpyxl import Workbook, load_workbook

# 新建
wb = Workbook()
ws = wb.active

ws['A1'] = '姓名'           # 写单个单元格
ws.append(['张三', 8000])    # 追加一行

wb.save('test.xlsx')

# 读取
wb = load_workbook('test.xlsx')
ws = wb.active
print(ws['A1'].value)        # 读单个单元格
```

### 2. 调格式（最常用）

```python
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

# 字体：加粗、颜色、大小
ws['A1'].font = Font(bold=True, color='FF0000', size=14)

# 填充：背景色
ws['A1'].fill = PatternFill(start_color='FFFF00', fill_type='solid')

# 对齐：居中、自动换行
ws['A1'].alignment = Alignment(horizontal='center', wrap_text=True)

# 边框
thin = Side(style='thin')
ws['A1'].border = Border(left=thin, right=thin, top=thin, bottom=thin)

# 数字格式：千分位、百分比、货币
ws['B2'].number_format = '#,##0.00'
ws['C2'].number_format = '0.00%'

# 列宽行高
ws.column_dimensions['A'].width = 20
ws.row_dimensions[1].height = 30
```

### 3. 合并单元格

```python
ws.merge_cells('A1:C1')   # A1 到 C1 合并成一个
ws['A1'] = '标题'
```

### 4. 公式

```python
ws['C2'] = '=A2+B2'        # 写公式，Excel 打开会算
ws['C3'] = '=SUM(A2:B2)'
```

### 5. 条件格式（高级，知道有就行）

```python
from openpyxl.formatting.rule import CellIsRule

# 大于 10000 的标红
red_fill = PatternFill(start_color='FFCCCC', fill_type='solid')
ws.conditional_formatting.add('B2:B100',
    CellIsRule(operator='greaterThan', formula=['10000'], fill=red_fill))
```

### 6. 图表（高级，知道有就行）

```python
from openpyxl.chart import BarChart, Reference

chart = BarChart()
data = Reference(ws, min_col=2, min_row=1, max_row=10)
chart.add_data(data, titles_from_data=True)
ws.add_chart(chart, 'D2')
```

---

## 接单时怎么用？（标准组合）

**场景：客户要"漂亮的 Excel 报表"**

```python
import pandas as pd
from openpyxl import load_workbook
from openpyxl.styles import Font, PatternFill, Alignment

# Step 1: pandas 处理数据（你熟悉的部分）
df = pd.read_csv('sales.csv')
result = df.groupby('月份').agg(总金额=('金额', 'sum')).reset_index()

# Step 2: pandas 快速导出（只要数据对）
result.to_excel('temp.xlsx', index=False)

# Step 3: openpyxl 调格式（让 AI 写这部分）
wb = load_workbook('temp.xlsx')
ws = wb.active

# 表头：加粗、蓝底白字、居中
for cell in ws[1]:
    cell.font = Font(bold=True, color='FFFFFF')
    cell.fill = PatternFill(start_color='366092', fill_type='solid')
    cell.alignment = Alignment(horizontal='center')

# 金额列：千分位格式
for row in ws.iter_rows(min_row=2, min_col=2, max_col=2):
    for cell in row:
        cell.number_format = '#,##0.00'

# 列宽自适应
for col in ws.columns:
    max_len = max(len(str(cell.value or '')) for cell in col)
    ws.column_dimensions[col[0].column_letter].width = max_len + 2

wb.save('销售月报.xlsx')
```

**你的角色**：Step 1 和 Step 2 你能看懂能改，Step 3 让 AI 写，你运行看效果，不满意让 AI 调。

---

## 什么时候需要 openpyxl？（判断标准）

| 客户需求 | 需要 openpyxl 吗？ |
|---------|------------------|
| "帮我把 CSV 转成 Excel" | 不需要，pandas 够了 |
| "Excel 报表要好看一点，表头有颜色" | **需要** |
| "按这个模板填数据"（发你一个 .xlsx） | **需要** |
| "Excel 里要有公式，打开能自动算" | **需要** |
| "数据超过 10 万行" | 不需要，pandas 更快 |

---

## 总结：你需要记住的

1. **openpyxl 是操作 Excel 的库**，能读写数据，但核心优势是**调格式、操作模板**
2. **pandas 和 openpyxl 是搭档**：pandas 处理数据，openpyxl 调格式
3. **你不用学 openpyxl 的细节**，知道它能干什么、什么时候用，具体代码让 AI 写
4. **判断标准**：客户要"好看的 Excel"、"按模板填"、"有公式"→ 上 openpyxl

完。
