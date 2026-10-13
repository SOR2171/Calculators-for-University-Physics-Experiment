# AGENTS.md

本文档为开发与维护此项目的 AI 代理（Agents）及协作者提供指引，详细阐述 `CUPE - 大物实验计算器` 项目中 `utils` 软件包的核心架构、数学与物理算法实现以及全流程数据处理规范。

---

## 1. 项目概览

- **项目定位**：大学物理实验（CUPE, College Physics Experiments）数据自动化计算与分析工具集。
- **核心目标**：提供符合《大学物理实验》教学大纲与国家计量规范（JJF 1059）的标准计算逻辑，杜绝精度丢失，严格执行“四舍六入五凑偶”修约。
- **目录结构**：
  - `python/utils/`：实验计算的核心工具包，承载数值处理、不确定度评定、统计拟合与格式化输出。
  - `python/*.ipynb`：各具体实验的数据处理 Notebook（如杨氏模量、霍尔效应、牛顿环等）。
  - `Excel/`：传统实验数据记录表，作为数据对照与参考。

---

## 2. utils 软件包全景架构

所有通用功能统一封装在 `python/utils/` 目录下，并通过 `python/utils/__init__.py` 导出常用接口：

```
python/utils/
├── __init__.py        # 统一对外暴露核心接口
├── formatting.py      # 有效数字科学修约、角度与弧分互转、精度推断
├── stats.py           # 统计分析：A/B类与合成不确定度、最小二乘法线性回归
└── io.py              # 交互式数据输入工具（CLI / 终端交互）
```

### 推荐导入方式
在各实验 Notebook 或脚本中，统一通过包路径显式导入：
```python
from python.utils import (
    scientific_round,
    calculate_stats,
    linear_regression,
    parse_angle_to_minutes,
    format_minutes_as_angle,
    get_decimal_places,
    input_decimal,
    input_data_group
)
```

---

## 3. utils 模块详解与数学实现

### 3.1 `formatting.py` —— 格式化与科学修约

#### `scientific_round(value, uncertainty)`
- **功能**：严格执行大学物理实验修约规则（**四舍六入五凑偶** / 银行家舍入法）：
  1. 不确定度 $u$ 优先修约，**保留 1 位有效数字**；
  2. 测量结果/均值末位与不确定度的末位对齐。
- **数学逻辑**：
  - 计算不确定度的最高有效位位置：$p = \lfloor \log_{10}(u) \rfloor$；
  - 构造对齐步长：$\text{prec} = 10^p$；
  - 使用 `decimal.Decimal.quantize(prec, rounding=ROUND_HALF_EVEN)` 分别对 $u$ 和 $\text{value}$ 修约。
- **入参**：
  - `value`: 数值或 `Decimal`（待修约的主值/平均值）。
  - `uncertainty`: 数值或 `Decimal`（合成绝对不确定度）。
- **返回值**：`tuple[Decimal, Decimal]` -> `(val_final, u_final)`。
- **异常处理**：若 `uncertainty <= 0`，返回原值字符串与 `"0"`。

#### `parse_angle_to_minutes(s)`
- **功能**：将分光计等实验中的角度读数字符串解析为系统内部统一单位：**分（arcminute，Decimal 类型）**。
- **支持输入格式**：
  - `'D.M'`：度.分格式（例如 `'120.15'` $\to 120^\circ 15' = 120 \times 60 + 15 = 7215'$）。**注意：小数点在此作为度与分的分隔符**。
  - `'D'`：纯度格式（例如 `'120'` $\to 120 \times 60 = 7200'$）。
  - `'D.M.S'`：度.分.秒格式（例如 `'120.15.30'` $\to 120 \times 60 + 15 + 30/60 = 7215.5'$）。
- **返回值**：`Decimal`（总分数）。

#### `format_minutes_as_angle(minutes, prec=None)`
- **功能**：将内部基准的分数还原并格式化为标准度分字符串 $D^\circ M'$。
- **入参**：
  - `minutes`: 总分数（数值或 `Decimal`）。
  - `prec`: 可选的 `Decimal` 步长（如通过修约计算得到的有效位对齐基准）。
- **返回值**：形如 `"-120°15'"` 或 `"120°15.3'"` 的字符串。

#### `get_decimal_places(data_strings)`
- **功能**：从原始输入的测量数据字符串列表中，获取小数点后的最大位数，用于表格展示时自适应保留输入仪器的原始分度精度。

---

### 3.2 `stats.py` —— 统计分析与回归拟合

#### `calculate_stats(data_list, delta_inst)`
- **功能**：计算单组多次直接测量数据的平均值、A 类、B 类及合成标准不确定度。
- **数学公式**：
  1. **算术平均值**：
     $$\bar{x} = \frac{1}{n} \sum_{i=1}^n x_i$$
  2. **Bessel 样本标准差**（$n > 1$）：
     $$s = \sqrt{\frac{1}{n-1} \sum_{i=1}^n (x_i - \bar{x})^2}$$
  3. **A 类标准不确定度**（有限次测量的偶然误差）：
     $$u_A = \frac{s}{\sqrt{n}}$$
  4. **B 类标准不确定度**（仪器极限误差 $\Delta_{\mathrm{inst}}$，按均匀分布置信因子 $k=\sqrt{3}$ 评定）：
     $$u_B = \frac{\Delta_{\mathrm{inst}}}{\sqrt{3}}$$
  5. **合成标准不确定度**：
     $$u_c = \sqrt{u_A^2 + u_B^2}$$
- **入参**：
  - `data_list`: 测量数据序列（支持 `list of Decimal`、`list of float`、`list of str`）。
  - `delta_inst`: 仪器允差/分度值误差限 $\Delta_{\mathrm{inst}}$（如游标卡尺 0.02 mm、千分尺 0.004 mm 等）。
- **返回值**：`tuple[Decimal, Decimal, Decimal, Decimal, Decimal]`
  - 依次为 `(mean, u_combined, u_a, u_b, s)`。

#### `linear_regression(x_list, y_list)`
- **功能**：最小二乘法一元线性拟合 $y = k \cdot x + b$，计算拟合参数、相关系数及斜率的不确定度。
- **数学公式**：
  - 均值 $\bar{x}, \bar{y}$ 与离差平方和：
    $$L_{xx} = \sum (x_i - \bar{x})^2, \quad L_{yy} = \sum (y_i - \bar{y})^2, \quad L_{xy} = \sum (x_i - \bar{x})(y_i - \bar{y})$$
  - 斜率与截距：
    $$k = \frac{L_{xy}}{L_{xx}}, \quad b = \bar{y} - k \cdot \bar{x}$$
  - 相关系数：
    $$r = \frac{L_{xy}}{\sqrt{L_{xx} \cdot L_{yy}}}$$
  - **斜率标准不确定度 $u(k)$**（当 $|r| < 1.0$ 且 $n > 2$ 时）：
    $$u(k) = |k| \sqrt{\frac{1/r^2 - 1}{n - 2}}$$
- **入参**：`x_list`, `y_list`（长度相等的数值序列）。
- **返回值**：`tuple[Decimal, Decimal, Decimal, Decimal]`
  - 依次为 `(k, b, r, u_k)`。

---

### 3.3 `io.py` —— 数据交互式采集

- `input_float(prompt, default=None)`：支持默认值与异常捕获的浮点数输入。
- `input_decimal(prompt, default=None)`：支持默认值与异常捕获的 `Decimal` 高精度输入。
- `input_int(prompt, default=None)`：支持默认值与异常捕获的整数输入。
- `input_data_group(name, count, unit="")`：针对多次重复测量的交互式输入，返回 `list[Decimal]`。

---

## 4. 大物实验数据处理规范

在为本项目编写或重构计算代码时，所有 Agent 必须遵守以下数据处理规范：

### 4.1 全链路高精度计算规范 (Decimal First)
1. **禁止过早引入浮点截断**：
   - 物理实验原始读数必须使用 `Decimal(str(raw))` 存储和运算，禁止使用 `Decimal(float(raw))`。
   - 中间计算（如求和、方差、乘积）必须尽可能全程保持 `Decimal` 或高位浮点数，**严禁在中间步骤手动四舍五入**。
2. **中间计算不修约，结果呈现才修约**：
   - 不确定度传递过程使用未经修约的原始数值进行方差合成；
   - 仅在输出最终展示结果（表格打印、摘要报告）时调用 `scientific_round`。

### 4.2 不确定度评定与传递规范
1. **直接测量**：
   - 必须同时考虑 A 类不确定度（统计离散度）与 B 类不确定度（仪器误差限）；
   - 默认置信概率取均匀分布（即 $u_B = \Delta_{\mathrm{inst}}/\sqrt{3}$），特殊说明除外。
2. **间接测量不确定度传递**：
   - 对幂乘积形式 $Y = C \cdot X_1^{p_1} \cdot X_2^{p_2} \cdots X_m^{p_m}$，优先计算各分量的相对不确定度平方和：
     $$u_r(Y) = \frac{u(Y)}{|Y|} = \sqrt{\sum_{i=1}^m \left( p_i \frac{u(X_i)}{X_i} \right)^2}$$
     再求合成绝对不确定度：$u(Y) = |Y| \cdot u_r(Y)$。
3. **逐差法规范**：
   - 遇到等间距线性增加的直接测量序列（如杨氏模量加减砝码标尺位移、声速驻波波腹坐标、牛顿环暗环读数），必须采用**逐差法**以充分利用数据并消除系统定值误差：
     $$\Delta x_j = x_{j + m} - x_j \quad (j = 1, \dots, m)$$
   - 逐差结果的均值作为有效位移量参与后续运算。

### 4.3 有效数字与修约规范
1. **修约法则**：严格执行“四舍六入五凑偶（ROUND_HALF_EVEN）”：
   - 尾数 $< 5$ 舍去；
   - 尾数 $> 5$ 进位；
   - 尾数 $= 5$ 时，若保留位末尾为奇数则进位，若为偶数则舍去。
2. **不确定度保留位数**：
   - 合成绝对不确定度 $u$ 统一保留 **1 位有效数字**。
3. **测量结果对齐规则**：
   - 最终物理量主值的最后一位必须与绝对不确定度 $u$ 的最后一位在相同数量级上严格对齐。
4. **最终结果展示格式**：
   - 带有数量级的量必须提取公因数，如：
     $$E = (2.05 \pm 0.08) \times 10^{11} \ \mathrm{Pa}$$
   - 相对不确定度以百分比形式表示，通常保留 1~2 位小数：
     $$u_r = 3.91\%$$

### 4.4 单位制管理规范
- **输入层**：保持实验仪器原始读数单位（例如毫米 $\mathrm{mm}$、微米 $\mu\mathrm{m}$、毫伏 $\mathrm{mV}$、分 $'$）。
- **计算层**：在代入物理公式前，必须显式转换为国际标准单位制（SI units: $\mathrm{m}, \mathrm{kg}, \mathrm{s}, \mathrm{A}, \mathrm{Pa}, \mathrm{T}$ 等），避免十进制数量级混淆。
- **展示层**：使用规范的物理量单位与常用工程科学计数法展现。

---

## 5. Notebook 实验编写标准模板

每个新编写的实验 Notebook（`python/<Experiment_Name>.ipynb`）建议遵循以下标准化 5 段结构：

```markdown
# 单元格 1: Markdown — 实验名称、原理说明与公式推导 (含 LaTeX)
# 单元格 2: Code — 依赖模块导入
import math
from decimal import Decimal
from python.utils import scientific_round, calculate_stats, linear_regression

# 单元格 3: Code — 实验参数、仪器允差与原始测量数据配置
raw_data = [Decimal("10.25"), Decimal("10.24"), ...]
delta_inst = Decimal("0.02")

# 单元格 4: Code — 核心数据处理 (中间计算、不确定度合成、逐差/拟合)
mean, u_c, u_a, u_b, s = calculate_stats(raw_data, delta_inst)

# 单元格 5: Code — 最终修约与规范化结果输出展示
val_final, u_final = scientific_round(mean, u_c)
print(f"测量结果: X = {val_final} ± {u_final} (单位)")
```

---

## 6. Agent 协作与代码维护要求

1. **外科手术式修改（Surgical Changes）**：
   - 当修复特定实验计算逻辑时，仅修改该 Notebook 对应的单元格，严禁无故重构或格式化相邻实验代码。
2. **向后兼容**：
   - `python.utils` 为所有 Notebook 的底层依赖，修改 `utils` 内的任何函数签名或修约逻辑前，必须验证现有所有 Notebook 的运行兼容性。
3. **无多余抽象（Simplicity First）**：
   - 保持工具函数纯净无副作用，不引入复杂的面向对象设计或冗余的类包装，优先使用函数式数据处理流程。
