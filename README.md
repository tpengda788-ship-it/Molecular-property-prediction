# Text-Embedding-Assisted Molecular Screening for Bi-Based Hybrid Halides

KNN-based literature-informed screening and ranking of organic cations using Contriever text embeddings.  
基于文献文本嵌入与 KNN 排序的有机阳离子筛选工具，用于预测刚度（Stiffness）与离子迁移激活能（Ion Migration Activation Energy）相关的语义相似度得分。

> **⚠️ 重要说明 / Important Note**  
> 本工具的输出为**相对语义相似度筛选得分**，不是经过物理标定的预测概率，也不代表分子刚度或离子迁移激活能的绝对预测值。得分的相对高低可用于候选分子的初步筛选排序，实验验证仍是必要步骤。  
> The output scores are **relative semantic-similarity screening descriptors**, not calibrated probabilities or quantitative predictions of physical properties. They serve as a literature-derived prior for candidate ranking; experimental validation remains essential.

---

## 项目概述 / Overview

本工具通过以下流程对候选有机阳离子进行筛选排序：

1. 使用预训练文本嵌入模型（Contriever）将分子描述转换为稠密向量
2. 从 `merged.json` 加载文献导出的高/低属性参考文本，构建参考集
3. 使用 KNN 分类器计算每个候选分子与高类参考文本的语义相似度得分
4. 输出排序结果至 Excel 表格，并生成得分分布直方图

**内部一致性验证**：对参考文本集进行了留一法（LOO）交叉验证（使用 Contriever 嵌入）：
- 刚度任务 LOO 准确率：**76.0%**（k=60，随机基线 55.7%，超出基线 +20.3 pp）
- 离子迁移激活能任务 LOO 准确率：**77.4%**（k=20，随机基线 53.8%，超出基线 +23.6 pp）

详见 `LOO_validation_complete.py` 和相关输出文件。

---

## 目录结构 / Directory Structure

```
.
├── predict.py                       # 主筛选脚本（运行此文件）
├── LOO_validation_complete.py       # LOO 交叉验证脚本
├── merged.json                      # 参考文本数据（从文献中提取）
├── Molecule_Information_90.xlsx     # 待筛选分子数据
├── requirements.txt                 # Python 依赖包
├── contriever/                      # Contriever 模型源代码
│   └── src/
│       ├── contriever.py
│       └── ...
└── output_<timestamp>/              # 自动生成的输出文件夹
    ├── Prediction_Results.xlsx      # 所有分子的筛选得分表格
    ├── Chart_Distribution.png       # 得分分布直方图
    ├── LOO_TFIDF_stiffness.csv      # TF-IDF LOO 结果（刚度）
    ├── LOO_TFIDF_activation_energy.csv
    ├── LOO_Contriever_stiffness.csv       # Contriever LOO 结果（刚度）
    ├── LOO_Contriever_activation_energy.csv
    ├── Origin_LOO_stiffness.csv           # Origin 绘图数据（刚度）
    ├── Origin_LOO_activation_energy.csv   # Origin 绘图数据（激活能）
    └── LOO_k_sensitivity.png              # LOO k 灵敏度图
```

---

## 环境要求 / Requirements

- Python 3.8+
- CUDA（推荐，非必需；默认可使用 CPU）

---

## 安装依赖 / Installation

```bash
pip install -r requirements.txt
```

---

## 模型准备 / Model Setup

### 选项 A：Contriever（推荐，默认 / Recommended）

代码将自动从 HuggingFace 下载 `akariasai/pes2o_contriever`，无需手动操作。

若网络受限（如中国大陆），可设置国内镜像：

```bash
export HF_ENDPOINT=https://hf-mirror.com
python predict.py
```

### 选项 B：本地缓存 / Local Cache

若模型已在本地缓存（`~/.cache/huggingface/hub/`），代码将自动加载，无需网络连接。

---

## 数据格式 / Data Format

### 参考文本数据（`merged.json`）

```json
{
  "all_description": {
    "high stiffness": ["Description of material 1...", "Description of material 2..."],
    "low stiffness":  ["Description of material 3..."],
    "high Ion migration activation energy of the crystal": ["..."],
    "low Ion migration activation energy of the crystal":  ["..."]
  }
}
```

参考文本数量 / Reference text counts：
| 类别 / Class | 数量 / Count |
|---|---|
| High stiffness | 148 |
| Low stiffness | 186 |
| High ion-migration activation energy | 86 |
| Low ion-migration activation energy | 100 |

### 待筛选分子数据（`Molecule_Information_90.xlsx`）

| 列名 / Column | 说明 / Description |
|---|---|
| `Final_ID` | 分子编号（若不存在则自动生成）|
| `Abbreviation` | 分子缩写名称 |
| `Description` | 分子的文本描述（用于嵌入计算）|

---

## 使用方法 / Usage

### 运行主筛选脚本

```bash
python predict.py
```

脚本将按以下步骤自动执行：

1. **加载模型**：初始化 Contriever 编码器
2. **加载数据**：读取参考文本与待筛选分子
3. **编码与排序**：对所有文本进行嵌入编码，使用 KNN（k=20~95，步长5）计算语义相似度得分并取均值
4. **保存结果**：输出 Excel 表格与分布图至带时间戳的文件夹

### 运行 LOO 交叉验证

```bash
python LOO_validation_complete.py
```

验证参考文本分类器的内部一致性，输出 k 灵敏度曲线图和 Origin 绘图数据。

---

## 输出说明 / Output Description

### `Prediction_Results.xlsx`

| Final_ID | Abbreviation | Stiffness_Score | Ion_Migration_Score |
|---|---|---|---|
| 1 | CHDA | 0.59 | 0.42 |
| 2 | HDA | 0.52 | 0.34 |
| ... | ... | ... | ... |

- **`Stiffness_Score`**：候选分子描述与高刚度参考文本的 KNN 语义相似度得分（0~1）
- **`Ion_Migration_Score`**：候选分子描述与高离子迁移激活能参考文本的 KNN 语义相似度得分（0~1）
- 得分反映候选分子与文献中高性能材料描述的**相对语义相似程度**，得分越高表示该分子描述在语义空间中更接近高性能参考文本
- **注意**：这些得分不是经过物理标定的绝对概率，不能直接用于定量预测分子刚度或离子迁移激活能数值

### `Chart_Distribution.png`

并排展示两项属性的预测得分分布直方图，用于快速评估候选分子整体分布形态和筛选阈值选取。

### `LOO_k_sensitivity.png`

留一法交叉验证的 k 灵敏度曲线，展示 Contriever 和 TF-IDF 两种嵌入方法在不同 k 值下的分类准确率，以及随机基线对比。

---

## 方法说明 / Methodology

本工具实现的是一个**文献导向的文本嵌入辅助筛选与排序工作流**，分两步：

**第一步（文献提取）**：使用 OpenScholar-8B 模型从文献语料库中提取描述高/低刚度和高/低离子迁移激活能材料的段落，构建参考文本集。

**第二步（嵌入排序）**：使用 Contriever 编码器将参考文本和待筛选分子描述转换为稠密向量，通过 KNN 相似度计算对候选分子排序。最终得分为 k=20~95（步长5）范围内高类得分的平均值。

**与传统机器学习的区别**：本工具不训练任何任务特定神经网络，不需要标注的分子属性数据集，输出为相对语义相似度排序而非定量物理性质预测。

---

## 常见问题 / FAQ

**Q1：`ModuleNotFoundError: No module named 'contriever'`**

请确保在项目根目录下运行脚本，且 `contriever/` 文件夹存在。

**Q2：模型下载失败 / Model download fails**

设置 HuggingFace 镜像后重试：
```bash
export HF_ENDPOINT=https://hf-mirror.com
```
或手动下载模型后将本地路径传入 `AutoModel.from_pretrained()`。

**Q3：`Some weights of BertModel were not initialized` 警告**

此为正常现象。Contriever 用于文本嵌入而非分类，pooler 层未初始化不影响嵌入质量，可安全忽略。

**Q4：`TOKENIZERS_PARALLELISM` 警告**

添加以下环境变量消除警告（不影响功能）：
```bash
export TOKENIZERS_PARALLELISM=false
```

**Q5：CUDA out of memory**

在脚本中将设备改为 CPU：
```python
device = 'cpu'
```

**Q6：LOO 验证运行很慢**

LOO 验证对 334 条参考文本（刚度任务）进行逐一交叉验证，在 CPU 上约需 10–20 分钟，GPU 上约 2–5 分钟，属正常现象。

---

## 引用 / Citation

如使用本工具或数据，请引用以下相关工作：

**本文**（预印本/正式发表后更新）：
```
Tong P., Yu C., Ouyang Y., et al. Text-Embedding-Assisted Design of Rigid Molecular 
Cations for Suppressing Ion Migration in Hybrid Single-Crystal X-ray Detectors.
The Journal of Physical Chemistry Letters, 2026.
```

**Contriever**：
```
Izacard G., et al. Unsupervised Dense Information Retrieval with Contrastive Learning.
arXiv:2112.09118, 2021.
```

**OpenScholar**：
```
Asai A., et al. OpenScholar: Synthesizing Scientific Literature with 
Retrieval-Augmented LMs. arXiv:2411.14199, 2024.
```

---

## 联系方式 / Contact

如有问题请联系：

- **通讯作者 / Corresponding authors**:
  - Lu Zhang: [luzhang@snnu.edu.cn](mailto:luzhang@snnu.edu.cn)
  - Jiaxue You: [jiaxuyou@cityu.edu.hk](mailto:jiaxuyou@cityu.edu.hk)
  - Shengzhong (Frank) Liu: [szliu@dicp.ac.cn](mailto:szliu@dicp.ac.cn)
- **代码问题 / Code issues**: [tpd2023@163.com](mailto:tpd2023@163.com)

---

## 许可证 / License

本项目遵循 MIT 许可证。Contriever 模型版权归原作者所有，请参阅 `contriever/` 目录下的许可证文件。

