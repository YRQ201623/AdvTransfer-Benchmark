# Image Caption Attack

针对图像描述（Image Captioning）和视觉语言模型（Vision-Language Model, VLM）的对抗攻击验证工具。本模块实现了对 BLIP 等图生文模型的对抗样本迁移攻击，评估对抗扰动对多模态模型生成文本的影响。

---

## 目录

- [文件说明](#文件说明)
- [环境依赖](#环境依赖)
- [快速开始](#快速开始)
- [BLIP 攻击验证](#blip-攻击验证)
- [攻击参数详解](#攻击参数详解)
- [输出结果解读](#输出结果解读)
- [支持模型](#支持模型)
- [常见问题](#常见问题)

---

## 文件说明

| 文件 | 功能 | 输入 | 输出 |
|------|------|------|------|
| `BLIP_verify.py` | 对 BLIP 图生文模型进行对抗攻击验证 | 原始图像 + 对抗样本 | 生成文本对比 / 攻击成功率 |

---

## 环境依赖

```bash
Python >= 3.8
PyTorch >= 1.12.0
torchvision >= 0.13.0
transformers >= 4.25.0
Pillow >= 9.0.0
numpy >= 1.21.0
tqdm >= 4.62.0
```

### 安装

```bash
pip install torch torchvision transformers Pillow numpy tqdm
```

> **注意：** BLIP 模型依赖 `transformers` 库，建议版本 >= 4.25.0 以确保 `BlipForConditionalGeneration` 接口兼容。

---

## 快速开始

### 单图像攻击验证

验证 FIA 攻击生成的对抗样本对 BLIP 模型生成描述的影响：

```bash
python BLIP_verify.py \
    --image-path ./data/sample.jpg \
    --adv-image-path ./data/sample_adv.jpg \
    --attack-method FIA \
    --model-type blip \
    --model-size base \
    --max-length 50
```

**输出示例：**

```
[Original Image]  Caption: "a cat sitting on a couch in a living room"
[Adversarial Image] Caption: "a blurry image of a dog running in the grass"
[Attack Success]    True (语义偏移检测)
```

### 批量验证

```bash
python BLIP_verify.py \
    --data-dir ./data/coco_val/ \
    --adv-dir ./data/coco_val_adv/ \
    --attack-method NAA \
    --batch-size 8 \
    --num-samples 500 \
    --output-dir ./caption_attack_results/
```

---

## BLIP 攻击验证

### 原理

图像描述模型（如 BLIP）通常以 CNN/ViT 作为视觉编码器，提取图像特征后通过文本解码器生成描述。对抗样本攻击通过向原始图像添加人眼不可见的扰动，使视觉编码器提取到**错误的特征表示**，导致解码器生成**语义偏离甚至完全错误的描述文本**。

本脚本对比以下两种输入的生成结果：

| 输入类型 | 说明 | 预期输出 |
|---------|------|---------|
| 原始图像 | 干净样本 | 准确描述图像内容的文本 |
| 对抗样本 | 添加扰动后的图像 | 语义偏移或错误的描述文本 |

### 攻击成功率判定

脚本提供两种判定方式：

#### 方式 1：语义相似度下降（推荐）

```bash
python BLIP_verify.py --metric semantic_similarity --threshold 0.3
```

- 使用 Sentence-BERT 计算原始描述与对抗描述之间的语义相似度
- 相似度 < 0.3 视为攻击成功

#### 方式 2：关键词匹配失败

```bash
python BLIP_verify.py --metric keyword_match --keywords "cat,sofa,room"
```

- 检测原始描述中的关键词是否在对搞描述中丢失
- 关键词丢失率 > 50% 视为攻击成功

---

## 攻击参数详解

### 基础参数

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `--image-path` | 必填（单图模式） | 原始图像路径 |
| `--adv-image-path` | 必填（单图模式） | 对抗样本路径 |
| `--data-dir` | 必填（批量模式） | 原始图像目录 |
| `--adv-dir` | 必填（批量模式） | 对抗样本目录 |
| `--attack-method` | FIA | 生成对抗样本的攻击方法 |
| `--model-type` | blip | 目标模型类型 |
| `--model-size` | base | 模型规模（base / large） |

### 生成参数

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `--max-length` | 50 | 生成文本最大长度 |
| `--num-beams` | 5 | Beam Search 束宽 |
| `--temperature` | 1.0 | 采样温度 |
| `--top-p` | 0.9 | Nucleus Sampling 参数 |

### 评估参数

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `--metric` | semantic_similarity | 评估指标类型 |
| `--threshold` | 0.3 | 攻击成功判定阈值 |
| `--keywords` | None | 关键词列表（逗号分隔） |
| `--batch-size` | 8 | 推理批大小 |

---

## 输出结果解读

### 单图模式输出

```
==================================================
Image: sample.jpg
Attack Method: FIA
--------------------------------------------------
Original Caption:
  "a cat sitting on a couch in a living room"

Adversarial Caption:
  "a blurry image of a dog running in the grass"

Semantic Similarity: 0.18  (< 0.30, Attack Success)
Keyword Match Rate:  0.00  (cat/sofa/room 全部丢失)
--------------------------------------------------
Attack Success: True
==================================================
```

### 批量模式输出

批量验证会生成 `results.json`：

```json
{
  "total_samples": 500,
  "attack_success_count": 423,
  "attack_success_rate": 0.846,
  "avg_semantic_similarity": 0.24,
  "method": "FIA",
  "target_model": "blip-base",
  "details": [
    {
      "image_id": "000001.jpg",
      "original": "a person riding a horse",
      "adversarial": "a statue of a person standing in a park",
      "similarity": 0.15,
      "success": true
    }
  ]
}
```

### 结果文件结构

```
caption_attack_results/
├── results.json              # 汇总统计
├── failed_cases/             # 攻击失败案例
│   ├── 000123.jpg
│   └── 000456.jpg
├── success_cases/            # 攻击成功案例（可视化对比）
│   ├── 000001_compare.png    # 原图 + 对抗图 + 描述对比
│   └── 000002_compare.png
└── report.html               # 可视化报告（浏览器打开）
```

---

## 支持模型

| 模型 | 类型 | 代码中标识 | 备注 |
|------|------|-----------|------|
| BLIP | 图生文 | `blip` | Salesforce 开源，支持条件/无条件生成 |
| BLIP-2 | 图生文 | `blip2` | 使用冻结 LLM，生成质量更高 |
| GIT | 图生文 | `git` | Microsoft 开源，生成速度较快 |

添加新模型：

```python
# 在 BLIP_verify.py 中注册
MODEL_REGISTRY = {
    'blip': 'Salesforce/blip-image-captioning-base',
    'blip2': 'Salesforce/blip2-opt-2.7b',
    'git': 'microsoft/git-base-coco',
}
```

---

## 常见问题

### Q1: 显存不足（OOM）怎么办？

BLIP-large 需要约 3GB 显存，BLIP-2 需要约 12GB。

```bash
# 使用 CPU 推理
python BLIP_verify.py --device cpu

# 或减小批大小
python BLIP_verify.py --batch-size 1

# 使用 BLIP-base 而非 large
python BLIP_verify.py --model-size base
```

### Q2: 对抗样本是从其他模型生成的，能迁移到 BLIP 吗？

可以。本脚本专门验证**迁移攻击**场景：

- 对抗样本在 ResNet-50 / VGG-16 等 CNN 上生成
- 直接输入到 BLIP（视觉编码器通常也是 CNN/ViT）
- 验证跨架构迁移性

```bash
# 对抗样本由 ResNet-50 上的 FIA 生成
# 测试对 BLIP 的迁移效果
python BLIP_verify.py \
    --adv-image-path ./adv_resnet50/fia/sample_adv.jpg \
    --attack-method FIA
```

### Q3: 如何生成对抗样本？

使用本仓库 `methods/` 目录下的攻击方法：

```bash
python ../methods/fia_pytorch.py \
    --image ./data/sample.jpg \
    --model resnet50 \
    --output ./adv/sample_adv.jpg
```

### Q4: 生成的描述全是乱码？

可能是对抗扰动过大，导致视觉编码器完全失效。尝试：

```bash
# 减小扰动预算
python BLIP_verify.py --eps 8  # 默认 16

# 或检查对抗样本是否归一化正确
python BLIP_verify.py --check-normalization
```

### Q5: 如何评估攻击对特定属性的影响？

使用属性级评估：

```bash
python BLIP_verify.py \
    --eval-attributes "object,action,scene" \
    --attribute-threshold 0.5
```

脚本会分别检测：
- **Object**（物体）：原图有"cat"，对抗后是否丢失
- **Action**（动作）：原图有"sitting"，对抗后是否变化
- **Scene**（场景）：原图有"living room"，对抗后是否改变

---

## 引用

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
| 2026-06-12 | v1.0.0 | 初始版本，支持 BLIP 图生文攻击验证 |

---

如有问题，请提交 [Issue](https://github.com/YRQ201623/AdvTransfer-Benchmark/issues)。
