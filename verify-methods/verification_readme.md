# 测试与验证文档 (Verification & Testing)

本文档介绍如何在 AdvTransfer-Benchmark 中运行测试、验证各攻击方法的迁移性能，并复现 Leaderboard 结果。

---

## 目录

- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [测试脚本说明](#测试脚本说明)
- [评估指标](#评估指标)
- [结果解读](#结果解读)
- [复现 Leaderboard](#复现-leaderboard)
- [常见问题](#常见问题)

---

## 环境要求

### 基础依赖

```bash
Python >= 3.8
PyTorch >= 1.12.0
torchvision >= 0.13.0
numpy >= 1.21.0
scipy >= 1.7.0
tqdm >= 4.62.0
```

### 安装

```bash
# 克隆仓库
git clone https://github.com/YRQ201623/AdvTransfer-Benchmark.git
cd AdvTransfer-Benchmark

# 安装依赖
pip install -r requirements.txt

# 下载预训练模型（可选，首次运行会自动下载）
python scripts/download_models.py
```

### 硬件要求

| 测试类型 | 最低配置 | 推荐配置 |
|---------|---------|---------|
| 单模型验证 | 1x GTX 1060 (6GB) | 1x RTX 3090 (24GB) |
| 全量迁移测试 | 1x RTX 3080 (10GB) | 2x RTX 3090 |
| Leaderboard 复现 | 2x RTX 3090 | 4x A100 (40GB) |

---

## 快速开始

### 1. 单方法单模型测试

验证 FIA 方法在 ResNet-50 上的攻击成功率：

```bash
python verify_single.py \
    --method FIA \
    --model resnet50 \
    --data-path ./data/imagenet_val \
    --eps 16 \
    --batch-size 32 \
    --num-samples 1000
```

**参数说明：**

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `--method` | 必填 | 攻击方法名，如 FIA, NAA, COPIA |
| `--model` | resnet50 | 源模型架构 |
| `--data-path` | ./data/imagenet_val | 验证数据集路径 |
| `--eps` | 16 | 扰动预算 (L∞, 像素值 0-255) |
| `--batch-size` | 32 | 批大小 |
| `--num-samples` | 1000 | 测试样本数 |

### 2. 迁移性测试

验证 FIA 从 ResNet-50 迁移到 VGG-16 的攻击成功率：

```bash
python verify_transfer.py \
    --method FIA \
    --source-model resnet50 \
    --target-model vgg16 \
    --data-path ./data/imagenet_val \
    --eps 16 \
    --batch-size 32
```

### 3. 一键运行全部测试

```bash
bash scripts/run_full_verification.sh
```

该脚本会自动：
1. 遍历所有支持的方法
2. 遍历所有源模型 → 目标模型组合
3. 生成 CSV 结果文件到 `./results/` 目录

---

## 测试脚本说明

### 脚本目录结构

```
scripts/
├── verify_single.py          # 单模型白盒攻击测试
├── verify_transfer.py        # 黑盒迁移攻击测试
├── verify_ensemble.py        # 集成模型攻击测试
├── run_full_verification.sh  # 全量自动化测试
├── evaluate_leaderboard.py   # Leaderboard 指标计算
└── download_models.py        # 预训练模型下载
```

### 核心脚本详解

#### `verify_single.py` — 单模型验证

测试指定方法在**白盒设定**下的攻击性能。

```bash
python verify_single.py \
    --method FIA \
    --model resnet50 \
    --attack-type pgd \
    --eps 16 \
    --steps 20 \
    --alpha 2.0
```

**输出示例：**

```
[2026-06-12 14:30:15] Method: FIA | Model: resnet50
[2026-06-12 14:30:15] Epsilon: 16/255 | Steps: 20 | Alpha: 2.0
Progress: 100%|████████████████████| 1000/1000 [02:15<00:00,  7.38it/s]

Results:
  Clean Accuracy:     85.20%
  Robust Accuracy:    62.30%  (PGD-20)
  Attack Success Rate: 37.70%
  Avg. Perturbation:  15.84/255 (L∞)
  Avg. Time:          0.135s/sample
```

#### `verify_transfer.py` — 迁移攻击验证

测试对抗样本从**源模型**到**目标模型**的迁移成功率。

```bash
python verify_transfer.py \
    --method FIA \
    --source-model resnet50 \
    --target-model vgg16 \
    --target-model densenet121 \
    --eps 16
```

**支持的目标模型列表：**

- `resnet18`, `resnet50`, `resnet101`, `resnet152`
- `vgg16`, `vgg19`
- `densenet121`, `densenet169`
- `inception_v3`, `inception_resnet_v2`
- `mobilenet_v2`, `efficientnet_b0`

#### `evaluate_leaderboard.py` — Leaderboard 指标计算

根据测试结果生成 Leaderboard 所需的各项指标：

```bash
python evaluate_leaderboard.py \
    --results-dir ./results \
    --output ./leaderboard.csv
```

输出 CSV 格式：

```csv
Method,Clean_Acc,PGD-10,PGD-20,PGD-50,AA,Time,Memory
FIA,85.2,62.3,58.1,54.2,51.8,120,4.2
NAA,84.1,59.8,56.2,52.1,49.5,90,3.8
...
```

---

## 评估指标

### 主要指标

| 指标 | 符号 | 说明 | 计算方式 |
|-----|------|------|---------|
| **Clean Accuracy** | $Acc_{clean}$ | 干净样本准确率 | 原始样本正确分类比例 |
| **Robust Accuracy** | $Acc_{robust}$ | 对抗样本准确率 | 对抗样本仍被正确分类的比例 |
| **Attack Success Rate** | ASR | 攻击成功率 | $1 - Acc_{robust}$ |
| **Transfer Rate** | TR | 迁移成功率 | 源模型生成对抗样本在目标模型上攻击成功的比例 |
| **AutoAttack (AA)** | $Acc_{AA}$ | 强攻击下的鲁棒准确率 | 使用 AutoAttack 评估 |

### 辅助指标

| 指标 | 说明 |
|-----|------|
| **Avg. Perturbation** | 平均扰动强度 (L∞ / L2) |
| **Avg. Time** | 单样本平均攻击耗时 |
| **Memory (GB)** | 峰值显存占用 |
| **GPU Hours** | 完成全量测试所需 GPU 时间 |

### 指标关系

```
Attack Success Rate = 1 - Robust Accuracy

Transfer Rate = (源模型生成对抗样本在目标模型上分类错误数) / 总样本数
```

---

## 结果解读

### 正常结果示例

```
Method: FIA
Source: resnet50 → Target: vgg16

Clean Acc:      85.20%  (正常范围: 80-90%)
PGD-10 Acc:     62.30%  (正常范围: 55-70%)
PGD-20 Acc:     58.10%  (正常范围: 50-65%)
AA Acc:         51.80%  (正常范围: 45-60%)
Transfer Rate:  91.50%  (高迁移性: >85%)
```

### 异常结果排查

| 现象 | 可能原因 | 解决方案 |
|-----|---------|---------|
| Clean Acc < 70% | 模型未加载预训练权重 | 检查 `--pretrained` 参数 |
| Robust Acc ≈ Clean Acc | 攻击步数/步长设置不当 | 增加 `--steps` 或调整 `--alpha` |
| Transfer Rate < 50% | 源模型与目标模型差异过大 | 尝试集成攻击或更换源模型 |
| OOM (显存不足) | 批大小过大 | 减小 `--batch-size` |
| 结果与 Leaderboard 偏差 > 5% | 随机种子不一致 | 设置 `--seed 42` |

---

## 复现 Leaderboard

### 完整复现流程

```bash
# 1. 准备环境
pip install -r requirements.txt

# 2. 下载 ImageNet 验证集（或子集）
# 将图片按类别放入 ./data/imagenet_val/n01440764/ 等目录

# 3. 运行全量测试（约需 4-6 小时，单卡 RTX 3090）
bash scripts/run_full_verification.sh

# 4. 生成 Leaderboard 表格
python evaluate_leaderboard.py \
    --results-dir ./results \
    --output ./leaderboard.csv

# 5. 可视化（可选）
python visualize.py --input ./leaderboard.csv --output ./figures/
```

### 快速验证（1000 样本子集）

如果无法使用完整 ImageNet 验证集，可使用 1000 样本快速验证：

```bash
python verify_transfer.py \
    --method FIA \
    --source-model resnet50 \
    --target-model vgg16 \
    --num-samples 1000 \
    --seed 42
```

> **注意：** 1000 样本子集结果与完整验证集可能存在 ±2% 的波动，仅供参考。

---

## 常见问题

### Q1: 如何添加新的测试方法？

在 `methods/` 目录下创建新的攻击文件（如 `my_attack.py`），实现以下接口：

```python
class MyAttack:
    def __init__(self, model, epsilon, steps, alpha):
        self.model = model
        self.eps = epsilon / 255.0
        self.steps = steps
        self.alpha = alpha / 255.0

    def forward(self, images, labels):
        # 返回对抗样本
        return adv_images
```

然后在 `verify_single.py` 中注册：

```python
from methods.my_attack import MyAttack

ATTACK_METHODS = {
    'FIA': FIAAttack,
    'NAA': NAAAttack,
    'MY_ATTACK': MyAttack,  # 新增
}
```

### Q2: 如何测试自己的模型？

修改 `models/` 目录下的模型加载逻辑，或直接在命令行指定：

```bash
python verify_transfer.py \
    --method FIA \
    --source-model path/to/your/model.pth \
    --model-arch resnet50 \
    --custom-model
```

### Q3: 结果保存到哪里？

默认保存路径：

```
results/
├── 20260612_143015/
│   ├── fia_resnet50_single.json      # 单模型结果
│   ├── fia_resnet50_vgg16.json       # 迁移结果
│   └── fia_resnet50_ensemble.json    # 集成结果
└── leaderboard.csv                   # 汇总表格
```

### Q4: 如何设置随机种子保证可复现？

所有脚本均支持 `--seed` 参数：

```bash
python verify_single.py --method FIA --seed 42
```

我们在 Leaderboard 中使用的默认种子为 **42**。

### Q5: 支持多 GPU 并行测试吗？

支持。使用 `torchrun` 或 `CUDA_VISIBLE_DEVICES`：

```bash
# 使用第 0,1 号 GPU
CUDA_VISIBLE_DEVICES=0,1 python verify_transfer.py --method FIA

# 或使用 DataParallel
python verify_transfer.py --method FIA --multi-gpu
```

---

## 引用

如果你在研究中使用了本 Benchmark 的测试框架，请引用：

```bibtex
@article{advtransfer2026,
  title={AdvTransfer Benchmark: A Fair Evaluation Framework for Adversarial Transferability},
  author={Your Name and Co-authors},
  journal={arXiv preprint},
  year={2026}
}
```

---

## 更新日志

| 日期 | 版本 | 更新内容 |
|-----|------|---------|
| 2026-06-12 | v1.0.0 | 初始版本，支持 FIA, NAA, COPIA 等方法 |

---

如有问题，请提交 [Issue](https://github.com/YRQ201623/AdvTransfer-Benchmark/issues) 或联系维护者。
