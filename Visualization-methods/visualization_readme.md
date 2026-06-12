# Visualization Methods

可视化方法目录，提供对抗样本攻击方法的可解释性分析工具，帮助理解攻击如何影响模型决策。

---

## 目录

- [文件说明](#文件说明)
- [环境依赖](#环境依赖)
- [快速开始](#快速开始)
- [CAM 可视化](#cam-可视化)
- [输出说明](#输出说明)
- [自定义配置](#自定义配置)

---

## 文件说明

| 文件 | 功能 | 输入 | 输出 |
|------|------|------|------|
| `CAM_visualization.py` | 类激活映射（Class Activation Mapping）可视化 | 原始图像 + 对抗样本 | 热力图叠加图像 |

---

## 环境依赖

```bash
Python >= 3.8
PyTorch >= 1.12.0
torchvision >= 0.13.0
numpy >= 1.21.0
matplotlib >= 3.5.0
opencv-python >= 4.5.0
Pillow >= 9.0.0
tqdm >= 4.62.0
```

### 安装

```bash
pip install torch torchvision numpy matplotlib opencv-python Pillow tqdm
```

---

## 快速开始

### 1. CAM 可视化

生成对抗样本前后的类激活映射对比图，直观展示攻击对模型关注区域的影响。

```bash
python CAM_visualization.py \
    --image-path ./data/sample.jpg \
    --model resnet50 \
    --attack-method FIA \
    --eps 16 \
    --target-layer layer4 \
    --output-dir ./visualization_results/
```

**参数说明：**

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `--image-path` | 必填 | 输入图像路径 |
| `--model` | resnet50 | 目标模型架构 |
| `--attack-method` | FIA | 攻击方法名（FIA / NAA / COPIA 等） |
| `--eps` | 16 | 扰动预算（L∞，像素值 0-255） |
| `--target-layer` | layer4 | 提取特征的层名 |
| `--output-dir` | ./results/ | 输出目录 |
| `--colormap` | jet | 热力图配色方案（jet / hot / coolwarm） |

---

## CAM 可视化

### 原理

Class Activation Mapping (CAM) 通过提取 CNN 最后一层卷积层的特征图，结合全连接层的权重，生成**类激活热力图**。热力图高亮区域表示模型在做出分类决策时最关注的图像区域。

### 攻击前后对比

```bash
python CAM_visualization.py \
    --image-path ./data/cat.jpg \
    --model resnet50 \
    --attack-method FIA \
    --eps 16 \
    --compare-mode
```

启用 `--compare-mode` 后，脚本会自动生成**三图对比**：

| 子图 | 内容 | 说明 |
|-----|------|------|
| (a) 原始图像 | 干净样本 + CAM 热力图 | 模型正常关注区域 |
| (b) 对抗样本 | 扰动图像 + CAM 热力图 | 攻击后模型关注区域偏移 |
| (c) 差异图 | 两热力图差值 | 红色 = 关注增强，蓝色 = 关注减弱 |

### 输出示例

```
visualization_results/
├── cam_original_cat.png          # 原始图像 CAM
├── cam_adversarial_cat.png       # 对抗样本 CAM
├── cam_comparison_cat.png      # 对比图（三合一）
└── cam_diff_cat.png              # 差异热力图
```

---

## 输出说明

### 单张图像输出

```bash
python CAM_visualization.py --image-path ./data/dog.jpg --model vgg16
```

生成文件：
- `cam_dog.jpg` — 热力图叠加原图
- 高亮区域越红，表示模型对该区域关注度越高
- 攻击成功后，高亮区域通常会发生明显偏移

### 批量可视化

```bash
python CAM_visualization.py \
    --data-dir ./data/imagenet_val/ \
    --num-samples 100 \
    --batch-size 16 \
    --attack-method NAA \
    --output-dir ./batch_vis/
```

批量模式下会生成：
- 每张图像的独立 CAM 图
- `summary.html` — 汇总页面，可浏览器查看所有结果
- `stats.json` — 关注区域偏移统计信息

---

## 自定义配置

### 更换目标层

不同模型的特征层命名不同：

| 模型 | 推荐 target-layer | 特征图尺寸 |
|------|------------------|-----------|
| ResNet-50 | `layer4` | 7×7 |
| VGG-16 | `features.29` | 14×14 |
| DenseNet-121 | `features.denseblock4` | 7×7 |
| Inception-v3 | `Mixed_7c` | 8×8 |

```bash
# VGG-16 示例
python CAM_visualization.py --model vgg16 --target-layer features.29

# DenseNet-121 示例
python CAM_visualization.py --model densenet121 --target-layer features.denseblock4
```

### 调整热力图透明度

```bash
python CAM_visualization.py --alpha 0.6  # 默认 0.5，范围 0-1
```

### 保存原始热力图（无叠加）

```bash
python CAM_visualization.py --save-raw-heatmap
```

---

## 可视化结果解读

### 正常情况（攻击前）

- 热力图高亮区域与图像语义对象一致
- 例如：猫图像的热力图集中在猫的身体/头部

### 攻击后（迁移攻击成功）

- 热力图高亮区域发生偏移
- 可能分散到背景区域或无关物体上
- 说明对抗样本误导了模型的注意力机制

### 攻击失败（防御成功）

- 热力图与原始图像基本一致
- 模型注意力未被扰动欺骗

---

## 引用

如果你在研究中使用本可视化工具，请引用：

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
| 2026-06-12 | v1.0.0 | 初始版本，支持 CAM 可视化 |

---

如有问题，请提交 [Issue](https://github.com/YRQ201623/AdvTransfer-Benchmark/issues) 或联系维护者。
