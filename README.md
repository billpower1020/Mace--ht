# MACE-HTMC

MACE-HTMC 是一个基于 Metropolis Monte Carlo 的结构搜索程序。程序随机交换指定子晶格中的元素，使用 MACE 对候选结构进行几何优化和能量预测，再根据 Metropolis 准则决定是否继承该候选结构。

当前示例体系为 `Sc-Sb-Te`，默认在 Sc/Sb 位点之间进行交换。

## 计算流程

```text
根目录 POSCAR
    -> 复制为 MCfolder/0/POSCAR
    -> MACE 优化，生成 CONTCAR 和能量
    -> 随机交换指定元素，生成下一步候选 POSCAR
    -> MACE 优化候选结构
    -> 根据能量差和温度进行 Metropolis 判定
    -> 接受候选或从上一个已接受结构重新生成候选
```

## 目录说明

```text
MACE HT/
|-- POSCAR                         # 无空位计算的初始结构
|-- 0.POSCAR                       # 空位模式使用的初始结构
|-- engine/runmetropolis.py        # 主要运行参数和启动入口
|-- model/mace_.py                 # MACE 加载、FIRE 优化和能量计算
|-- MCobjects/Metropolis/          # Metropolis 接受判据及主循环
|-- cores/stepObject.py            # 计算步和结构状态管理
|-- generateNewStructure/          # 随机交换原子、生成候选结构
|-- calculators/                   # VASP 计算接口
`-- MCfolder/                      # 默认计算输出目录
```

## 环境安装

建议使用独立的 Conda 或 venv 环境。当前已验证环境为 Python 3.11，主要软件版本包括 `mace-torch 0.3.16`、`torch 2.11.0`、`ase 3.28.0` 和 `pymatgen 2026.5.4`。

安装主要依赖：

```powershell
python -m pip install mace-torch ase pymatgen numpy prettytable
```

PyTorch 的 CPU/CUDA 安装方式取决于计算机和显卡环境。首次使用尚未缓存的 MACE 基础模型时可能需要联网下载模型。

## 输入结构

普通计算读取根目录的 `POSCAR`。当前文件是标准 VASP POSCAR 格式，包含：

```text
Sc Sb Te
5  49 81
Direct
```

即 5 个 Sc、49 个 Sb 和 81 个 Te，共 135 个原子。

当 `vac_dope = False` 时，程序把 `POSCAR` 复制到 `MCfolder/0/POSCAR`。当 `vac_dope = True` 时，改为读取 `0.POSCAR`。

注意：`0.POSCAR` 中的 `V` 在普通化学元素语义下代表钒。只有程序完成空位占位符的删除与转换后，才能把它作为空位使用，不能直接送给 MACE 当作空位计算。

## 运行参数

主要参数位于 `engine/runmetropolis.py`：

| 参数 | 当前值 | 含义 |
|---|---:|---|
| `OUTPUT_ROOT` | `MCfolder` | 所有 MC 步骤的输出根目录 |
| `num_loops` | `10` | Monte Carlo 循环步数 |
| `T` | `444` | Metropolis 温度，单位 K |
| `random_seed` | `2026` | Python、NumPy、PyTorch 随机种子 |
| `sublattice_symbols_lst` | `[["Sc", "Sb"]]` | 允许互换的元素组 |
| `elements_str_for_vaspkit` | `"Sc Sb Te"` | 输出结构的元素排列顺序 |
| `load_model` | `"mace"` | 能量模型，可选 `mace`、`mattersim`、`chgnet` 或 `None` |
| `load_path` | `"medium"` | MACE 基础模型名称或本地模型路径 |
| `from_contcar` | `True` | 从已接受结构的优化后 `CONTCAR` 产生下一候选 |
| `vac_dope` | `False` | 是否启用空位模式 |
| `exchange_times` | `1` | 每个候选结构交换的原子对数 |

修改初始结构、步数、温度和交换元素后，从项目根目录运行：

```powershell
python engine/runmetropolis.py
```

也可以使用模块方式运行：

```powershell
python -m engine.runmetropolis
```

## 输出文件

一次 10 步计算通常产生：

```text
MCfolder/
|-- steps.log
|-- 0/
|   |-- POSCAR
|   |-- CONTCAR
|   `-- relaxation_output.txt
|-- 1/
|   |-- POSCAR
|   |-- CONTCAR
|   |-- relaxation_output.txt
|   |-- Accept.txt
|   `-- time_record.txt
|-- ...
`-- 11/POSCAR
```

各文件含义：

- `POSCAR`：该编号步骤生成的候选结构。
- `CONTCAR`：候选结构经过 MACE/FIRE 优化后的结构。
- `relaxation_output.txt`：使用的 MACE 模型和最终总能量，能量单位为 eV。
- `Accept.txt`：本步候选结构的 Metropolis 接受概率。
- `time_record.txt`：本步模型计算耗时，单位为秒。
- `steps.log`：总尝试步数和接受次数汇总。
- `process.log`：随机交换的元素及原子索引。

编号文件夹保存的是候选结构，不代表该候选一定被接受。当前实现会提前生成下一候选，因此完成 10 步后出现只有 `POSCAR` 的 `11/` 文件夹属于现有程序行为。

## Metropolis 接受判据

设当前已接受结构能量为 `E1`，候选结构能量为 `E2`：

- `E2 < E1`：候选结构必然接受，概率为 1。
- `E2 >= E1`：接受概率为 `exp(-(E2-E1)/(kB*T))`，再与随机数比较。

因此 `Accept.txt` 中小于 1 的数值只是接受概率，不等同于最终已经接受。当前文件还没有记录随机数和明确的 `accepted=True/False` 状态。

## 可复现计算

`random_seed = 2026` 会固定 Python 的原子选择、NumPy 的 Metropolis 随机数以及 PyTorch 随机状态。改变该数值可以获得另一条独立 MC 轨迹。

要复现同一轨迹，还必须同时满足：

- 使用完全相同的初始 `POSCAR`、参数、模型和软件环境。
- 从干净的新输出目录开始，不混用以前计算的 `CONTCAR` 或能量文件。
- 使用相同的 CPU/GPU 类型；部分 GPU 运算仍可能有微小浮点差异。

重新开始计算时，推荐修改 `OUTPUT_ROOT` 为新的目录名，例如 `MCfolder_seed2026`。程序发现目标 `0/POSCAR` 已存在时不会重新复制根目录 `POSCAR`，MACE 发现已有 `CONTCAR` 时也会复用旧结果。

## Windows 注意事项

`generateNewStructure/exchangeAtoms.py` 中仍包含外部命令：

```text
vaspkit
mv POSCAR POSCAR_PRI
mv POSCAR_REV POSCAR
```

Windows `cmd` 默认没有 `mv`，未安装或未配置 VASPKIT 时也找不到 `vaspkit`，因此终端可能显示“不是内部或外部命令”，并且不会生成 `POSCAR_PRI`。候选 `POSCAR` 在调用这些命令前已经由 pymatgen 写出，所以 MACE 主流程通常仍能继续，但 VASPKIT 的重排步骤没有实际执行。

## 当前限制

- `Accept.txt` 只保存接受概率，没有保存最终接受结果。
- `exchanged.txt` 当前写在接受前的旧结构目录，不能直接用来判断同编号候选是否被接受。
- `load=True` 的续算路径尚未完整验证，不建议直接用于重要长任务。
- 重复使用同一个 `MCfolder` 会复用旧输出，可能造成新旧计算数据混合。

正式长时间计算前，建议先在新的输出目录运行 5 到 10 步，检查结构、能量、接受概率和耗时是否合理，再增大 `num_loops`。
