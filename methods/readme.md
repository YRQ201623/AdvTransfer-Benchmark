# Methods

This directory contains the implementation of all supported adversarial transferability methods.

| Method | File | Paper | Venue |
|--------|------|-------|-------|
| FIA | [fia_pytorch.py](fia_pytorch.py) | [Feature Importance-aware Transferable Adversarial Attacks](https://openaccess.thecvf.com/content/ICCV2021/papers/Wang_Feature_Importance-Aware_Transferable_Adversarial_Attacks_ICCV_2021_paper.pdf) | ICCV 2021 |
| NAA | [naa_pytorch.py](naa_pytorch.py) | ... | ... |
| COPIA | [copia_pytorch.py](copia_pytorch.py) | ... | ... |

## Usage

```python
from fia_pytorch import FIAAttack

# Example usage
attack = FIA_Attack(model, epsilon=16/255, ...)
adv = attack(image, label)
