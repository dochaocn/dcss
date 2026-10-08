---
name: xlsx
description: 当电子表格文件是主要输入或输出时使用本技能。包括：打开、读取、编辑或修复现有 .xlsx、.xlsm、.csv、.tsv（如增列、计算公式、格式、图表、清洗杂乱数据）；从零或用其他数据源新建表格；或在表格类格式之间转换。尤其当用户随口提到「下载里的那个 xlsx」等并希望对文件做处理或基于其产出时触发。也适用于将杂乱表格数据整理成规范电子表格。交付物必须是表格文件。当主要交付物是 Word、HTML 报告、独立 Python 脚本、数据库管道或 Google Sheets API 集成时，不要使用本技能，即使涉及表格数据。
license: 专有许可，完整条款见 LICENSE.txt
---

# 输出要求

## 所有 Excel 文件

### 专业字体
- 除非用户另有说明，交付物应使用一致的专业字体（如 Arial、Times New Roman）

### 零公式错误
- 每个 Excel 模型交付时必须**零**公式错误（#REF!、#DIV/0!、#VALUE!、#N/A、#NAME?）

### 保留现有模板（更新模板时）
- 修改文件时须研究并**精确**匹配现有格式、样式与惯例
- 勿对已有固定模式的文件强行套用标准化格式
- 已有模板惯例**始终优先**于本指南

## 财务模型

### 颜色规范
除非用户或现有模板另有说明：

#### 行业通用颜色约定
- **蓝字（RGB: 0,0,255）**：硬编码输入、用户会改动以做情景分析的数值  
- **黑字（RGB: 0,0,0）**：**所有**公式与计算  
- **绿字（RGB: 0,128,0）**：从同一工作簿其他工作表链接的数据  
- **红字（RGB: 255,0,0）**：指向其他文件的外部链接  
- **黄底（RGB: 255,255,0）**：需关注的关键假设或待更新单元格  

### 数字格式规范

#### 必守规则
- **年份**：格式为文本字符串（如 `"2024"`，不要显示为 `2,024`）  
- **货币**：使用 `$#,##0`；**务必**在表头标明单位（如「收入（百万美元）」）  
- **零值**：用数字格式将零显示为 `-`，百分比同理（如 `"$#,##0;($#,##0);-"`）  
- **百分比**：默认一位小数 `0.0%`  
- **倍数**：估值倍数格式为 `0.0x`（如 EV/EBITDA、P/E）  
- **负数**：用括号 `(123)`，不要用前导负号 `-123`  

### 公式构建规则

#### 假设放置
- **所有**假设（增长率、利润率、倍数等）放在独立假设单元格  
- 公式中用单元格引用，不要硬编码数值  
- 示例：用 `=B5*(1+$B$6)` 而不是 `=B5*1.05`  

#### 避免公式错误
- 核对所有单元格引用是否正确  
- 注意范围 off-by-one  
- 确保各预测期公式一致  
- 用边界情况测试（零、负数）  
- 避免无意循环引用  

#### 硬编码文档要求
- 在旁注单元格或表格末注释。格式：`"来源：[系统/文档]，[日期]，[具体引用]，[若有 URL]"`  
- 示例：  
  - `"来源：公司 10-K，FY2024，第 45 页，收入附注，[SEC EDGAR URL]"`  
  - `"来源：公司 10-Q，2025 Q2，附件 99.1，[URL]"`  
  - `"来源：Bloomberg Terminal，2025/8/15，AAPL US Equity"`  
  - `"来源：FactSet，2025/8/20，一致预期界面"`  

# XLSX 创建、编辑与分析

## 概述

用户可能要求创建、编辑或分析 .xlsx 文件。不同任务适用不同工具与工作流。

## 重要要求

**公式重算需要 LibreOffice**：可假定已安装 LibreOffice，并通过 `scripts/recalc.py` 重算公式。脚本首次运行会自动配置 LibreOffice，包括在沙箱环境 Unix 套接字受限时（由 `scripts/office/soffice.py` 处理）。

## 读取与分析数据

### 用 pandas 做数据分析
对数据分析、可视化与基础操作，使用 **pandas**：

```python
import pandas as pd

# 读 Excel
df = pd.read_excel('file.xlsx')  # 默认：第一张表
all_sheets = pd.read_excel('file.xlsx', sheet_name=None)  # 全部表为字典

# 分析
df.head()      # 预览
df.info()      # 列信息
df.describe()  # 统计

# 写 Excel
df.to_excel('output.xlsx', index=False)
```

## Excel 文件工作流

## 关键：用公式，勿硬编码计算结果

**始终用 Excel 公式，而不要在 Python 里算好再写入数值。** 这样表格在数据变更时仍可重算。

### ❌ 错误 — 硬编码计算结果
```python
# 差：在 Python 中求和再写入
total = df['Sales'].sum()
sheet['B10'] = total  # 硬编码 5000

# 差：在 Python 算增长率
growth = (df.iloc[-1]['Revenue'] - df.iloc[0]['Revenue']) / df.iloc[0]['Revenue']
sheet['C5'] = growth  # 硬编码 0.15

# 差：在 Python 算平均
avg = sum(values) / len(values)
sheet['D20'] = avg  # 硬编码 42.5
```

### ✅ 正确 — 使用 Excel 公式
```python
# 好：由 Excel 求和
sheet['B10'] = '=SUM(B2:B9)'

# 好：增长率为 Excel 公式
sheet['C5'] = '=(C4-C2)/C2'

# 好：用 Excel 函数求平均
sheet['D20'] = '=AVERAGE(D2:D19)'
```

**所有**计算（合计、百分比、比率、差值等）均适用。源数据变化时表格应能重算。

## 常见工作流
1. **选工具**：数据操作用 pandas，公式/格式用 openpyxl  
2. **创建/加载**：新建工作簿或加载已有文件  
3. **修改**：增删数据、公式、格式  
4. **保存**：写入文件  
5. **重算公式（若使用公式则必须）**：运行 `scripts/recalc.py`  
   ```bash
   python scripts/recalc.py output.xlsx
   ```
6. **校验并修复错误**：  
   - 脚本返回含错误详情的 JSON  
   - 若 `status` 为 `errors_found`，查看 `error_summary` 中的类型与位置  
   - 修复后再次重算  
   - 常见错误：  
     - `#REF!`：无效引用  
     - `#DIV/0!`：除零  
     - `#VALUE!`：类型错误  
     - `#NAME?`：无法识别的函数名  

### 新建 Excel 文件

```python
# 使用 openpyxl 编写公式与格式
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

wb = Workbook()
sheet = wb.active

# 数据
sheet['A1'] = 'Hello'
sheet['B1'] = 'World'
sheet.append(['Row', 'of', 'data'])

# 公式
sheet['B2'] = '=SUM(A1:A10)'

# 格式
sheet['A1'].font = Font(bold=True, color='FF0000')
sheet['A1'].fill = PatternFill('solid', start_color='FFFF00')
sheet['A1'].alignment = Alignment(horizontal='center')

# 列宽
sheet.column_dimensions['A'].width = 20

wb.save('output.xlsx')
```

### 编辑已有 Excel 文件

```python
from openpyxl import load_workbook

# 加载
wb = load_workbook('existing.xlsx')
sheet = wb.active  # 或 wb['SheetName']

# 多表
for sheet_name in wb.sheetnames:
    sheet = wb[sheet_name]
    print(f"Sheet: {sheet_name}")

# 修改单元格
sheet['A1'] = 'New Value'
sheet.insert_rows(2)  # 在第 2 行前插入
sheet.delete_cols(3)  # 删除第 3 列

# 新表
new_sheet = wb.create_sheet('NewSheet')
new_sheet['A1'] = 'Data'

wb.save('modified.xlsx')
```

## 重算公式

openpyxl 创建或修改的文件中，公式以字符串形式存在，但未必含计算结果。请使用提供的 `scripts/recalc.py` 重算：

```bash
python scripts/recalc.py <excel_file> [timeout_seconds]
```

示例：
```bash
python scripts/recalc.py output.xlsx 30
```

脚本会：
- 首次运行时自动配置 LibreOffice 宏  
- 重算所有工作表中公式  
- 扫描**所有**单元格中的 Excel 错误（#REF!、#DIV/0! 等）  
- 返回含详细错误位置与数量的 JSON  
- 在 Linux 与 macOS 上均可工作  

## 公式验证清单

快速确保公式正确：

### 必备验证
- [ ] **抽测 2–3 个引用**：在搭建完整模型前确认拉取的是正确数值  
- [ ] **列映射**：确认 Excel 列对应正确（如第 64 列 = BL，不是 BK）  
- [ ] **行偏移**：Excel 行为 1 起算（DataFrame 第 5 行 = Excel 第 6 行）  

### 常见陷阱
- [ ] **NaN**：用 `pd.notna()` 检查空值  
- [ ] **最右列**：财年数据常在 50 列以后  
- [ ] **多处匹配**：搜索所有出现，不要只取第一处  
- [ ] **除零**：在公式中用 `/` 前检查分母（#DIV/0!）  
- [ ] **错误引用**：确认引用指向预期单元格（#REF!）  
- [ ] **跨表引用**：链接工作表时使用正确格式（`Sheet1!A1`）  

### 公式测试策略
- [ ] **从小做起**：先对 2–3 个单元格测公式再铺开  
- [ ] **检查依赖**：公式中引用的单元格都存在  
- [ ] **边界情况**：包含零、负数、极大值  

### 解读 scripts/recalc.py 输出
脚本返回 JSON，例如：
```json
{
  "status": "success",           // 或 "errors_found"
  "total_errors": 0,
  "total_formulas": 42,
  "error_summary": {
    "#REF!": {
      "count": 2,
      "locations": ["Sheet1!B5", "Sheet1!C10"]
    }
  }
}
```

## 最佳实践

### 库选择
- **pandas**：数据分析、批量操作、简单导出  
- **openpyxl**：复杂格式、公式、Excel 专有功能  

### 使用 openpyxl
- 单元格索引为 1 基（row=1, column=1 即 A1）  
- 读取计算结果用 `data_only=True`：`load_workbook('file.xlsx', data_only=True)`  
- **警告**：以 `data_only=True` 打开再保存，公式会被数值**永久替换**  
- 大文件：只读用 `read_only=True`，只写用 `write_only=True`  
- 公式会保留但不会自动求值 — 用 `scripts/recalc.py` 更新显示值  

### 使用 pandas
- 指定 dtype 避免推断问题：`pd.read_excel('file.xlsx', dtype={'id': str})`  
- 大文件只读部分列：`pd.read_excel('file.xlsx', usecols=['A', 'C', 'E'])`  
- 日期：`pd.read_excel('file.xlsx', parse_dates=['date_column'])`  

## 代码风格准则
**重要**：生成用于 Excel 操作的 Python 代码时：
- 代码尽量短、简洁，避免多余注释  
- 避免冗长变量名与冗余操作  
- 避免不必要的 print  

**对 Excel 文件本身**：
- 复杂公式或重要假设可在单元格加注释  
- 硬编码数值注明数据来源  
- 关键计算与模型区块加说明备注  
