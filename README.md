# Model_Pro_pipeline

# Seismic Data Preprocessing Pipeline / 地震数据预处理流

本项目包含一套完整的数据处理流程，旨在将原始的 MiniSEED 地震波形数据转换为适用于深度学习模型的标准化 HDF5 数据集。处理流程主要分为三个阶段：**格式转换与预处理**、**数据标注**以及**数据切片**。

## 📋 流程概览 (Pipeline Overview)

| 阶段 | 脚本文件 | 功能描述 | 输入/输出 |
| --- | --- | --- | --- |
| **Step 1** | `mseed2hdf5pro.py` | **格式转换 (Conversion)**<br>

<br>读取三分量 mseed 数据，去除仪器响应，去噪，并转换为 HDF5 格式。 | `.mseed` + XML → `.hdf5` (Raw) |
| **Step 2** | `Labeling.py` | **标签生成 (Labeling)**<br>

<br>根据事件目录，在 HDF5 文件中生成点对点 (Point-wise) 的软标签 (0-1)。 | `.hdf5` (Raw) → `.hdf5` (Labeled) |
| **Step 3** | `Cut_data_pro_5min.py` | **数据切片 (Segmentation)**<br>

<br>将连续数据切割为固定长度的时间窗，对正负样本采用不同的采样策略。 | `.hdf5` (Labeled) → `.hdf5` (Segments) |

---

## 🛠️ 详细处理步骤

### Step 1: 格式转换与预处理 (Conversion)

**脚本**: `mseed2hdf5pro.py`

该步骤负责将原始的地震台站数据（按年/台站/分量存储的 `.mseed` 文件）整合为单一的 HDF5 文件。

* **核心功能**:
1. **多分量合并**: 自动匹配同一时间段的 `EHZ`, `EHN`, `EHE` 三个分量。
2. **信号预处理**:
* 去除仪器响应 (Remove Instrument Response, Output=DISP)。
* 去线性趋势 (Detrend Linear) 和 去均值 (Demean)。


3. **元数据注入**: 将采样率、台站名及**初步事件信息** (根据 `events.txt` 匹配) 写入 HDF5 属性中。


* **输入依赖**:
* 原始 mseed 文件夹结构。
* `events.txt`: 包含事件时间范围的目录文件。
* Station XML 文件 (`.xml`): 用于去除仪器响应。



---

### Step 2: 软标签生成 (Labeling)

**脚本**: `Labeling.py`

读取 Step 1 生成的 HDF5 文件，根据元数据中的事件时间生成对应的标签数组 (`labels`)。

* **标签策略 (Soft Labeling)**:
为了让模型更好地学习地震事件的起始和结束，采用渐进式标签:
* **Event (1)**: 事件持续期间，标签值为 `1`。
* **Noise (0)**: 非事件期间，标签值为 `0`。
* **Transition (Ramp)**:
* **Fade-in**: 事件开始前 60秒内，标签从 `0` 线性增加到 `1`。
* **Fade-out**: 事件结束后 60秒内，标签从 `1` 线性减少到 `0`。





---

### Step 3: 数据切片与增强 (Segmentation)

**脚本**: `Cut_data_pro_5min.py`

将长时间的连续波形切割为固定长度（例如 5 分钟）的样本，用于模型训练。针对正负样本不平衡问题，采用了动态重叠策略。

* **切片逻辑**:
1. **正样本 (Events)**:
* 当检测到 `labels > 0` 时，触发**高重叠采样**。
* **重叠率**: 步长 (Step Size) 设置为窗口长度的 20% (即 80% 重叠)，以此增加正样本数量 (Data Augmentation)。


2. **负样本 (Noise)**:
* 当数据段内无事件时，采用**无重叠采样**。
* 步长等于窗口长度，避免产生过多冗余的负样本。




* **输出内容**:
每个切片被保存为独立的 Group，包含 `data` (3通道波形), `labels` (点标签), `binary_label` (二分类标签)。

---

## 📂 目录结构示例

```text
Project_Root/
├── Raw_Data/                  # 原始输入
│   └── 2020/
│       └── Station_A/
│           ├── EHZ/...mseed
│           ├── EHN/...mseed
│           └── EHE/...mseed
├── Processed_HDF5/            # Step 1 & 2 输出
│   └── JJG.Station_A.ALL.2020.136.hdf5 (包含 'data' 和 'labels')
└── Cut_Data_Segments/         # Step 3 最终输出
    └── JJG.Station_A.ALL.2020.136_all_segments.hdf5
        ├── segment_1/
        │   ├── data (30000, 3)
        │   └── labels (30000,)
        ├── segment_2/
        └── ...

```

## 🚀 快速开始

```bash
# 1. 运行格式转换 (需修改脚本中的路径配置)
python mseed2hdf5pro.py

# 2. 生成标签
python Labeling.py

# 3. 执行数据切片
python Cut_data_pro_5min.py

```

---

`multi_watershed_preprocessor_v3.py` 是一个高度集成的处理器，它直接读取分类好的波形数据（mseed/sac），执行重采样、去噪，并根据台站数量自动调整切片策略，最终生成用于深度学习的 **HDF5 数据集**。

它实际上合并并优化了旧流程中“格式转换”和“数据切片”的步骤，是一个 **End-to-End** 的预处理方案。

---

## STEP 4: 统一预处理与切片 (Unified Preprocessing & Segmentation)

运行脚本 `multi_watershed_preprocessor_v3.py`。该脚本负责将经过 Step 1 和 Step 2 整理好的连续波形数据，转换为模型可直接读取的 HDF5 切片文件。

### 1. 核心功能 (Key Features)

该脚本 (v3版本) 针对多流域、多格式数据进行了深度适配：

* **多格式兼容**:
* 标准格式: `Net.Sta.Comp.Year.DOY.mseed`
* 川藏线格式: `YYYYMMDDHH_Station_Comp_merged.mseed`
* SAC 格式: `Station...Comp.sac`


* **统一采样率**: 自动将所有数据重采样至 **100Hz**。
* **智能切片策略 (Adaptive Segmentation)**:
* **单台站模式 (Single Station)**: 当某流域某年只有一个台站时，采用 **6分钟 (360s)** 切片窗口。
* **多台站模式 (Multi Station)**: 当存在多个台站组网时，采用 **5分钟 (300s)** 切片窗口，以便于进行网络协同分析。


* **高效存储**: 每天的所有切片存储在一个单一的 `.h5` 文件中，避免生成数百万个小文件。

### 2. 运行脚本

使用命令行运行脚本，可以通过参数灵活配置输入输出路径和处理模式。

#### 基本运行

```bash
python multi_watershed_preprocessor_v3.py \
  --source "E:\JJG_SORTED_DATA_2025_NORMALIZED" \
  --output "E:\JJG_HDF5_DATASET" \
  --parallel

```

#### 参数说明

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `--source` | (代码内默认路径) | Step 2 生成的标准化数据根目录 |
| `--output` | (代码内默认路径) | HDF5 数据集的输出目录 |
| `--parallel` | False | 启用多进程并行处理 (推荐开启，大幅提升速度) |
| `--group-by` | `station` | 输出目录结构。`station`: 按台站分文件夹; `year`: 按年份分文件夹 |
| `--dry-run` | False | 仅扫描并打印统计信息，不实际生成文件 (用于检查数据) |
| `--segment-multi` | 300.0 | 多台站模式下的窗口长度 (秒) |
| `--segment-single` | 360.0 | 单台站模式下的窗口长度 (秒) |

### 3. 处理逻辑详解

脚本内部执行流程如下：

1. **扫描 (Scanning)**: 遍历目录，自动识别流域 (Watershed)、年份和台站，并建立文件索引。
2. **分组 (Grouping)**: 将同一天、同一台站的不同分量（Z/N/E）文件聚合。如果包含小时级文件（如川藏线数据），会自动按时间顺序排序。
3. **加载与预处理 (Loading & Preprocessing)**:
* 合并分段波形。
* **去趋势 (Detrend)**: 去除线性趋势和均值。
* **重采样 (Resample)**: 统一至 100Hz。


4. **切片 (Segmentation)**:
* 根据 `--overlap` (默认 80%) 计算步长。
* 滑动窗口截取数据。


5. **写入 (Writing)**: 将切片数据写入 HDF5，结构如下：

```text
Output_HDF5_File.h5
├── /Attributes (Global Metadata: Station, Year, SamplingRate...)
├── /Watershed_Year_DOY_Station_seg000
│   ├── data_Z (Array: 30000 points for 5min)
│   ├── data_N
│   ├── data_E
│   └── Attributes (Start_Time, Segment_Index)
├── /Watershed_Year_DOY_Station_seg001
└── ...

```

### 4. 输出目录结构

脚本运行完成后，`output` 目录下将生成如下结构的数据集（以默认 `--group-by station` 为例）：

```text
E:\JJG_HDF5_DATASET
└─Jiangjiagou
    └─2025
        └─453007897
            ├─Jiangjiagou_2025_163_453007897_ALL.h5  (包含该台站当天的所有切片)
            ├─Jiangjiagou_2025_164_453007897_ALL.h5
            └─...

```

### 5. 依赖环境

除了之前的依赖外，本步骤需要：

* `h5py`: 用于读写 HDF5 文件
* `pandas`: 用于生成处理日志映射表
* `obspy`: 核心波形处理

```bash
pip install h5py pandas obspy

```

---

通过运行 `run_manager.py`，我们将成百上千个分散的 HDF5 文件索引到一个 **SQLite 数据库**中，并生成符合模型输入要求的“样本对”（Sample Pairs）。这使得 PyTorch `DataLoader` 可以高效地查询和加载数据，而无需在训练/预测时遍历文件系统。

---

## STEP 5: 数据库构建与样本配对 (Database Construction)

运行脚本 `run_manager.py`。该脚本调用核心库 `universal_debris_flow_database_manager_v2.py`，完成数据的索引、逻辑配对和数据集划分，最终生成一个 `.db` 数据库文件。

### 1. 核心概念 (Core Concepts)

该管理器解决了泥石流监测中**单台站**与**多台站**数据不统一的问题：

* **索引 (Indexing)**: 自动扫描 Step 4 生成的 HDF5 文件，读取元数据（采样率、时长、是否包含事件等）存入 SQLite `files` 表。
* **配对 (Pairing)**: 模型需要成对的输入（上游/下游），管理器根据台站数量自动选择策略:
* **多台站 (Physical Pairing)**: 物理上的上游台站 vs 下游台站。
* **单台站 (Virtual Self-Pairing)**: 将同一台站的数据在时间上错开（例如前 5 分钟作为“上游”，后 5 分钟作为“下游”，中间重叠 4 分钟），从而利用单台站数据驱动需要双输入的模型。


* **数据集划分 (Splitting)**: `run_manager.py` 默认将所有数据标记为 `predict` (预测集)，用于模型推理。

### 2. 配置运行

打开 `run_manager.py`，根据实际情况修改配置区域：

```python
# run_manager.py

# ================= 配置区域 =================
# 输入：Step 4 生成的 HDF5 文件夹路径
DATA_ROOT = r"/path/to/JJG_HDF5_DATASET"

# 输出：生成的 SQLite 数据库文件名
DB_PATH = "debris_flow_predict.db"

# 通道配置：根据模型需求选择 ['Z'] 或 ['Z', 'N', 'E']
CHANNELS = ['Z'] 

# 采样步长：预测时设为 1 (不跳过任何数据)
SKIP_STEP = 1
# ===========================================

```

### 3. 运行脚本

```bash
python run_manager.py

```

### 4. 运行过程解析

脚本执行时会经历以下阶段：

1. **初始化**: 清理旧的 `.db` 文件，建立新的数据库 Schema (包含 `files`, `segments`, `sample_pairs` 表)。
2. **索引 (Indexing)**: 遍历 `DATA_ROOT` 下的所有流域文件夹，将文件路径和元数据写入数据库。
3. **生成配对 (Generating Pairs)**:
* 对于单台站年份，执行 `_generate_single_virtual_pairs`。
* 对于多台站年份，执行 `_generate_multi_physical_pairs`。


4. **标记预测集**: 强制执行 SQL 更新，将所有样本的 `dataset_split` 设置为 `'predict'`，确保它们能被推理代码读取。

### 5. 输出结果

运行完成后，目录下会生成一个 SQLite 文件（例如 `debris_flow_predict.db`）。
此数据库包含模型所需的所有索引信息：

* **`sample_pairs` 表**: 每一行代表一个可以直接输入模型的样本，包含：
* `upstream_file_path`, `downstream_file_path`
* `upstream_slice_start`, `downstream_slice_start` (数据切片位置)
* `pair_type` (virtual_single / physical)



### 6. 在模型中使用 (Usage in PyTorch)

在您的深度学习预测代码中，无需手动读取文件，只需使用管理器提供的 Dataset 类：

```python
from universal_debris_flow_database_manager_v2 import UniversalDebrisFlowDatabaseManager, DebrisFlowPairDataset
from torch.utils.data import DataLoader

# 1. 连接数据库
manager = UniversalDebrisFlowDatabaseManager('debris_flow_predict.db')

# 2. 创建 Dataset (指定读取 'predict' 集)
dataset = DebrisFlowPairDataset(
    db_manager=manager, 
    watershed='Jiangjiagou', 
    split='predict', 
    components=['Z']
)

# 3. 创建 DataLoader
loader = DataLoader(dataset, batch_size=32, shuffle=False)

# 4. 迭代数据进行预测
for batch in loader:
    upstream = batch['upstream']     # Tensor: [Batch, 1, 30000]
    downstream = batch['downstream'] # Tensor: [Batch, 1, 30000]
    # output = model(upstream, downstream)

```
