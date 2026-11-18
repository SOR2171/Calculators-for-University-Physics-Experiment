# AGENTS.md

本文档为开发与维护此项目的 AI 代理（Agents）及协作者提供全局指引，详细阐述 `CUPE - 大物实验计算器` 项目中 `utils` 软件包的核心架构、数学与物理算法实现、不确定度评定准则以及全流程数据处理规范。

---

## 1. 项目概览

- **项目定位**：大学物理实验（CUPE, College Physics Experiments）数据自动化计算与分析工具集。
- **核心目标**：提供符合《大学物理实验》教学大纲与国家计量规范（JJF 1059）的标准计算逻辑，杜绝浮点精度丢失，严格执行“四舍六入五凑偶”修约。
- **目录结构**：
  - `python/utils/`：实验计算的核心工具包，承载数值处理、不确定度评定、统计拟合与格式化输出。
  - `python/*.ipynb`：各具体实验的数据处理 Notebook（如杨氏模量、迈克尔逊干涉仪、粘滞系数、温度传感器等）。
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
在各实验 Notebook 或脚本中，统一通过包路径显式按需导入：
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
  1. 不确定度 $u$ 优先修约，**统一保留 1 位有效数字**；
  2. 测量结果/均值末位与不确定度的末位在相同数量级上严格对齐。
- **数学逻辑**：
  - 计算不确定度的最高有效位位置：$p = \lfloor \log_{10}(u) \rfloor$；
  - 构造对齐步长：$\mathrm{prec} = 10^p$（若 $u$ 进位至 $10 \times 10^p$，则步长自动进位为 $10^{p+1}$）；
  - 使用 `decimal.Decimal.quantize(prec, rounding=ROUND_HALF_EVEN)` 分别对 $u$ 和 $\mathrm{value}$ 修约。
- **入参**：
  - `value`: 数值、字符串或 `Decimal`（待修约的主值/平均值）。
  - `uncertainty`: 数值、字符串或 `Decimal`（合成绝对不确定度）。
- **返回值**：`tuple[Decimal, Decimal]` $\to$ `(val_final, u_final)`。
- **异常处理**：若 `uncertainty <= 0`，返回原值字符串与 `"0"`。

#### `parse_angle_to_minutes(s)`
- **功能**：将分光计等光学实验中的角度读数字符串解析为系统内部统一单位：**分（arcminute，`Decimal` 类型）**。
- **支持输入格式**：
  - `'D.M'`：度.分格式（例如 `'120.15'` $\to 120^\circ 15' = 120 \times 60 + 15 = 7215'$）。**注意：小数点在此作为度与分的分隔符，而非十进制小数**。
  - `'D'`：纯度格式（例如 `'120'` $\to 120 \times 60 = 7200'$）。
  - `'D.M.S'`：度.分.秒格式（例如 `'120.15.30'` $\to 120 \times 60 + 15 + 30/60 = 7215.5'$）。
- **返回值**：`Decimal`（总弧分数）。

#### `format_minutes_as_angle(minutes, prec=None)`
- **功能**：将内部基准的总弧分数还原并格式化为标准度分字符串 $D^\circ M'$。
- **入参**：
  - `minutes`: 总弧分数（数值或 `Decimal`）。
  - `prec`: 可选的 `Decimal` 步长（如通过修约计算得到的有效位对齐基准）。
- **返回值**：形如 `"-120°15'"` 或 `"120°15.3'"` 的字符串。

#### `get_decimal_places(data_strings)`
- **功能**：从原始输入的测量数据字符串列表中获取小数点后的最大位数，用于表格展示时自适应保留仪器的原始分度精度。

---

### 3.2 `stats.py` —— 统计分析与回归拟合

#### `calculate_stats(data_list, delta_inst)`
- **功能**：计算单组多次直接测量数据的算术平均值、A 类、B 类及合成标准不确定度。
- **数学公式**：
  1. **算术平均值**：
     $$\bar{x} = \frac{1}{n} \sum_{i=1}^n x_i$$
  2. **Bessel 样本标准差**（$n > 1$）：
     $$s = \sqrt{\frac{1}{n-1} \sum_{i=1}^n (x_i - \bar{x})^2}$$
  3. **A 类标准不确定度**（有限次测量的随机误差）：
     $$u_A = \frac{s}{\sqrt{n}}$$
  4. **B 类标准不确定度**（仪器极限误差 $\Delta_{\mathrm{inst}}$，默认按均匀分布置信因子 $k=\sqrt{3}$ 评定）：
     $$u_B = \frac{\Delta_{\mathrm{inst}}}{\sqrt{3}}$$
  5. **合成标准不确定度**：
     $$u_c = \sqrt{u_A^2 + u_B^2}$$
- **入参**：
  - `data_list`: 测量数据序列（支持 `list[Decimal]`、`list[float]`、`list[str]`）。
  - `delta_inst`: 仪器允差/分度值误差限 $\Delta_{\mathrm{inst}}$（如游标卡尺 $0.02\,\mathrm{mm}$、千分尺 $0.004\,\mathrm{mm}$ 等；若不计入 B 类误差则传 `Decimal("0")`）。
- **返回值**：`tuple[Decimal, Decimal, Decimal, Decimal, Decimal]`
  - 依次为 `(mean, u_combined, u_a, u_b, s)`。

#### `linear_regression(x_list, y_list)`
- **功能**：最小二乘法一元线性拟合 $y = k \cdot x + b$，计算斜率、截距、线性相关系数及斜率标准不确定度。
- **数学公式**：
  - 均值 $\bar{x}, \bar{y}$ 与离差平方和：
    $$L_{xx} = \sum_{i=1}^n (x_i - \bar{x})^2, \quad L_{yy} = \sum_{i=1}^n (y_i - \bar{y})^2, \quad L_{xy} = \sum_{i=1}^n (x_i - \bar{x})(y_i - \bar{y})$$
  - 斜率与截距：
    $$k = \frac{L_{xy}}{L_{xx}}, \quad b = \bar{y} - k \cdot \bar{x}$$
  - 线性相关系数：
    $$r = \frac{L_{xy}}{\sqrt{L_{xx} \cdot L_{yy}}}$$
  - **斜率标准不确定度 $u(k)$**（当 $|r| < 1.0$ 且 $n > 2$ 时）：
    $$u(k) = |k| \sqrt{\frac{1/r^2 - 1}{n - 2}}$$
- **入参**：`x_list`, `y_list`（等长数值序列）。
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

在为本项目编写或重构计算代码时，所有 Agent 必须遵守以下核心规范：

### 4.1 全链路高精度计算规范 (Decimal First)
1. **禁止过早引入浮点截断**：
   - 物理实验原始读数与常量必须使用字符串初始化 `Decimal("...")`，**严禁**使用 `Decimal(float_val)`。
   - 涉及开方等超越运算需借助 `math.sqrt` 时，必须遵循 `Decimal(str(math.sqrt(float(expr))))` 的桥接模式，将浮点转换隔离在最小局部。
2. **中间计算不修约，结果呈现才修约**：
   - 中间统计量、逐差均值及不确定度传递过程必须保留完整 `Decimal` 精度进行方差合成；
   - 仅在最终结果展示层调用 `scientific_round`。

### 4.2 不确定度评定与传递规范
1. **直接测量不确定度**：
   - 同时考虑 A 类不确定度（统计离散度 $u_A = s/\sqrt{n}$）与 B 类不确定度（仪器误差限 $u_B = \Delta_{\mathrm{inst}}/\sqrt{3}$）；
   - 单次测量物理量（$n=1$）无 A 类分量，其标准不确定度直接取 $u = u_B = \Delta_{\mathrm{inst}}/\sqrt{3}$。
2. **间接测量不确定度传递**：
   - 对幂乘积形式 $Y = C \cdot X_1^{p_1} \cdot X_2^{p_2} \cdots X_m^{p_m}$，优先计算各分量的相对不确定度平方和：
     $$u_r(Y) = \frac{u(Y)}{|Y|} = \sqrt{\sum_{i=1}^m \left( p_i \frac{u(X_i)}{X_i} \right)^2}$$
     再求合成绝对不确定度：$u(Y) = |Y| \cdot u_r(Y)$。
   - 对加减线性组合形式 $Y = \sum c_i X_i$，按绝对不确定度方和根合成：$u(Y) = \sqrt{\sum (c_i u(X_i))^2}$。
3. **逐差法（Successive Differences）规范**：
   - 遇到等间距线性变化的直接测量序列 $x_0, x_1, \dots, x_{2M-1}$（如杨氏模量砝码拉伸、迈克尔逊干涉仪条纹计数、驻波波腹位置），必须对半分为前后两组（跨度 $M$）执行逐差：
     $$\Delta x_i = x_{i+M} - x_i \quad (i = 0, 1, \dots, M-1)$$
   - **逐差均值与 A 类不确定度**：
     $$\overline{\Delta x} = \frac{1}{M} \sum_{i=0}^{M-1} \Delta x_i, \quad s(\Delta x) = \sqrt{\frac{1}{M-1} \sum_{i=0}^{M-1} (\Delta x_i - \overline{\Delta x})^2}, \quad u_A(\overline{\Delta x}) = \frac{s(\Delta x)}{\sqrt{M}}$$
   - **逐差法 B 类不确定度的两种标准口径**：
     - **口径 A（常规宏观标尺直接评定，默认）**：适用于光杠杆标尺、米尺等常规测量，直接以仪器单次误差限评定 $u_B(\overline{\Delta x}) = \frac{\Delta_{\mathrm{inst}}}{\sqrt{3}}$（如《杨氏模量》）。
     - **口径 B（独立读数线性组合传递）**：适用于精密光学测微手轮（如《迈克尔逊干涉仪》），由于 $\overline{\Delta x} = \frac{1}{M}\sum (x_{i+M} - x_i)$ 由 $2M$ 个独立读数线性组合而成，按误差传递定律取 $u_B(\overline{\Delta x}) = \sqrt{\frac{2}{M}} \frac{\Delta_{\mathrm{inst}}}{\sqrt{3}}$。
4. **一元线性回归补充规范**：
   - 标准斜率不确定度直接使用 `linear_regression` 返回的 `u_k`。
   - 若实验明确要求评定**截距 $b$ 的标准不确定度 $u(b)$**，在代码中按以下解析式补充计算：
     $$u(b) = u(k) \cdot \sqrt{\frac{1}{n} \sum_{i=1}^n x_i^2} = u(k) \cdot \sqrt{\bar{x}^2 + \frac{L_{xx}}{n}}$$

### 4.3 有效数字与修约规范（铁律）
1. **修约法则**：严格执行“四舍六入五凑偶（`ROUND_HALF_EVEN`）”。
2. **不确定度保留位数（禁止越权）**：
   - 合成绝对不确定度 $u$ **一律调用 `scientific_round` 保留 1 位有效数字**。
   - **严禁** Agent 自行手写“首位为 1 或 2 时保留 2 位有效数字”的分支逻辑，确保全项目修约行为与 `python.utils.scientific_round` 绝对一致。
3. **测量结果对齐规则**：
   - 最终物理量主值的末位必须与绝对不确定度 $u$ 的末位在相同数量级上严格对齐。
   - **严禁伪精度违规**：若 $u$ 修约到个位（如 $u = 2\,\mathrm{nm}$），主值必须同步修约到个位（如 $\lambda = 642 \pm 2\,\mathrm{nm}$），**绝对禁止**写成 $641.7 \pm 2\,\mathrm{nm}$。
4. **最终结果展示格式**：
   - 带有较大或较小数量级的物理量必须先除以特征数量级（如 $10^{11}$）再送入 `scientific_round` 修约，提取公因数展示：
     $$E = (2.05 \pm 0.08) \times 10^{11}\,\mathrm{Pa}$$
   - 相对不确定度 $u_r$ 与相对误差 $E_r$ 以百分比形式展示，统一保留 2 位小数（如 `u_r = 0.23%`）。

### 4.4 单位制管理规范
- **输入层**：保持实验讲义或原始记录表中的仪器自然读数单位（如 $\mathrm{mm}$、$\mathrm{cm}$、$\mathrm{g}$、$\mathrm{V}$、$^\circ\mathrm{C}$）。
- **计算层**：代入物理定律前，必须显式转换为国际标准单位制（SI units: $\mathrm{m}, \mathrm{kg}, \mathrm{s}, \mathrm{A}, \mathrm{K}, \mathrm{Pa}$ 等），并在变量命名中体现单位后缀（如 `L_m`, `mean_lambda_nm`, `i_ua`），杜绝十进制数量级混淆。
- **展示层**：使用规范的物理量单位与 ASCII 表格对齐打印。

### 4.5 间接多工况测量与不确定度评定避坑指南（重点红线）
1. **严禁凭空臆造仪器误差限**：
   - 对于非单一仪器直接读数的间接计算量（如多普勒效应测声速、透镜焦距等），若讲义未明确给出单一仪器误差限，**严禁凭空捏造 $\Delta_{\mathrm{inst}}$**。
2. **严禁盲目将单点微积分传递误差套给多工况总均值**：
   - 在多工况（如不同转速、不同载荷、不同温度点）下多次测量并间接求解某一物理量时，单次读数的分辨力限制已在多工况样本标准差 $s$ 与 A 类不确定度 $u_A = s/\sqrt{n}$ 中客观体现。
   - **禁止**强行对各点传递的 $u_B$ 求平均并叠加到总均值上，否则会导致合成不确定度异常膨胀并抹杀有效数字精度。
3. **多工况均值评定标准**：
   - 统一采用直接统计评定 $u = u_A = s/\sqrt{n}$，调用 `scientific_round` 将均值末位与 $u_A$ 末位对齐。

---

## 5. Notebook 实验编写标准模板

每个实验 Notebook（`python/<Experiment_Name>.ipynb`）应遵循以下标准化结构，并在条件允许时预填典型实测数据与运行输出（`outputs`）：

```python
# 单元格 1: Markdown — 实验标题、仪器读数说明、实验原理与完整 LaTeX 公式推导
# 单元格 2: Code — 依赖模块导入与加载确认提示
import math
from decimal import Decimal
from python.utils import scientific_round, calculate_stats, linear_regression

print("XX实验计算模块加载完成。")

# 单元格 3: Markdown + Code — 物理常数、仪器误差限 Δ_inst 与原始测量数据配置 (直接在列表中配置 Decimal，无需交互式 input 阻塞运行)
raw_data = [Decimal("10.25"), Decimal("10.24"), Decimal("10.26"), Decimal("10.25"), Decimal("10.23")]
delta_inst = Decimal("0.02")

# 单元格 4: Markdown + Code — 中间数据处理表格打印 (使用 "=" 和 "-" 分隔线对齐列) 与统计/拟合/逐差计算
mean, u_c, u_a, u_b, s = calculate_stats(raw_data, delta_inst)

# 单元格 5: Code — 最终科学修约与规范化实验报告结果汇总输出
val_final, u_final = scientific_round(mean, u_c)
print("=" * 50)
print(f"★ 最终规范修约结果 : X = {val_final} ± {u_final} (单位)")
print("=" * 50)
```

---

## 6. Agent 协作与代码维护要求

1. **严禁使用 Git 相关命令（No Git Rule）**：
   - 严禁调用任何 `git` 命令（包括但不限于 `git status`, `git log`, `git diff`, `git commit`, `git checkout`, `git push` 等）。所有文件读写、对比与检查工作仅限通过文件系统与代码编辑工具完成。
2. **外科手术式修改（Surgical Changes）**：
   - 修复或更新特定实验计算逻辑时，仅修改目标 Notebook 对应的单元格，严禁无故重构、重命名或格式化其他无关实验文件。
3. **向后兼容（Backward Compatibility）**：
   - `python.utils` 为所有 Notebook 的核心底层依赖，修改 `utils` 内的任何函数签名或修约逻辑前，必须确保现有全部 Notebook 的调用兼容性。
4. **无多余抽象（Simplicity First）**：
   - 保持工具函数纯净无副作用，Notebook 内优先采用直观的顺序式或轻量级函数式数据处理流程，不引入复杂的面向对象设计或冗余类封装。
5. **非阻塞式数据配置（Non-Blocking Notebook Execution）**：
   - 在 Jupyter Notebook 中，优先将原始测量数据作为 `Decimal` 列表/元组直接定义在数据输入单元格内，方便用户直接修改并一键运行全部单元格，避免在 Notebook 中使用阻塞式的 `input()` 弹窗。
6. **成品纯净性原则（No Agent Confusion in Production Artifacts）**：
   - **严禁**将 Agent 思考过程中产生的试错、争论、反面教材或“方案一 vs 方案二”的混乱对比写入用户的实验 Notebook 成品中。
   - 交付的 Notebook 即为最终标准实验报告，必须通篇保持公式准确、逻辑唯一、排版整洁。