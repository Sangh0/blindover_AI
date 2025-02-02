# Quantize the weights of model to achieve faster inference speed

- Models: ShuffleNetV2, ResNet18 and ResNet50
- Comparison size of models based on 33 classes (KB):

    |                 | ShuffleNetV2 (x0.5) | ResNet18 | ResNet50 |
    | :-------------: | :-----------------: | :------: | :------: |
    |  Normal Model   |       1629.99       | 44843.61 | 94597.31 |
    | Quantized Model |       684.15        | 11402.92 | 24540.52 |
    |     factor      |        2.38         |   3.93   |   3.85   |

- Comparison inference speed of models on our test set (seconds for 152 images):
- We did not quantize MobileNetV3 and EfficientNetV2

    - The experiments were conducted on a **I9-12900K CPU**

    |                                  | ShuffleNetV2 (x0.5) | ResNet18 | ResNet50 | EfficientNetV2 | MobileNetV3 |
    | :------------------------------: | :-----------------: | :------: | :------: | :------------: | :---------: |
    | Inference speed of Float32 Model |       4.249s        | 141.720s | 377.158s |   498.407s     |   4.574s    |
    |    Accuracy of Float32 Model     |       93.42%        |  93.42%  |  94.08%  |     94.08%     |    96.05%   |
    |      Inference speed of QAT      |       2.838s        |  2.893s  |  3.646s  |       -        |      -      |
    |         Accuracy of QAT          |       94.08%        |  96.71%  |  98.03%  |       -        |      -      |
    |      Inference speed of PTQ      |       2.815s        |  3.007s  |  4.204s  |       -        |      -      |
    |         Accuracy of PTQ          |       92.11%        |  93.42%  |  93.42%  |       -        |      -      |


    - The experiments were conducted on a **virtual server**


        <details><summary> <b> The specifications of our virtual server </b> </summary>
    
        - Inter(R) Xeon(R) CPU @ 2.20GHz
        - Thread(s) per core: 2
        - Core(s) per socket: 1
        - Socket(s) : 1
        - Stepping: 0
        - BogoMIPS: 4399.99
    
        </details>


    |                                  | ShuffleNetV2 (x0.5) | ResNet18 | ResNet50 | EfficientNetV2 | MobileNetV3 |
    | :------------------------------: | :-----------------: | :------: | :------: | :------------: | :---------: |
    | Inference speed of Float32 Model |       206.832s      |2864.013s |8032.249s |      -         |   157.911s  |
    |    Accuracy of Float32 Model     |        93.42%       |  93.42%  |  94.08%  |      -         |   96.05%    |
    |      Inference speed of QAT      |       154.552s      |245.029s  | 351.875s |      -         |     -       |
    |         Accuracy of QAT          |        94.08%       | 96.71%   |  98.03%  |      -         |     -       |
    |      Inference speed of PTQ      |       152.067s      |239.481s  | 352.040s |      -         |     -       |
    |         Accuracy of PTQ          |        92.11%       | 93.42%   |  93.42%  |      -         |     -       |


### Process Guide for Quantization

- Explanation step by step with simple example codes

<details><summary> <b>First Method: PTQ (Post Training Quantization)</b> </summary>

```
step 1. Loading a model that include QuantStub() and DeQuantStub() from torch.quantization
```

```python
import torch.nn as nn
from torch.quantization import QuantStub, DeQuantStub

class Model(nn.Module):
    def __init__(self, model):
        self.model = model
        self.quant = QuantStub()
        self.dequant = DeQuantStub()

    def forward(self, x):
        x = self.quant(x)
        x = self.model(x)
        x = self.dequant(x)
        return x

model = Model()
```

```
step 2. Training model or loading pre-trained weight
```

```python
# training model
training(model, train_loader)

# OR loading pre-trained weight
model.load_state_dict(torch.load('pretrained_weight.pt'))
```

```
step 3. Fusing modules such as nn.Conv2d and nn.BatchNorm2d
```

```python
import torch

def fuse_modules(model):
    model = model.cpu()
    model.eval()
    modules = [
        ['conv1', 'bn1'],
    ]

    return torch.quantization.fuse_modules(model, modules)

fused_model = fuse_modules(model)
```

```
step 4. Preparing quantization for quantizable model with float32 bit
```

```python
import torch

def prepare_ptq(model, backend='x86'):
    model.train()
    model = model.cpu()
    model.qconfig = torch.quantization.get_default_qconfig(backend)
    return torch.quantization.prepare(model)

prepared_model = prepare_ptq(fused_model)
```

```
step 5. Data calibrilation
```

```python
import torch

def calibration(model, data_loader, device=torch.device('cpu')):
    model.eval()
    model = model.to(device)
    with torch.no_grad():
        for image, _ in data_loader:
            image = image.to(device)
            _ = model(image)

calibration(prepared_model)
```

```
step 6. Converting the weight from float32 to uint8 for model and saving the quantized weight
```

```python
import torch

def converting(model):
    model.eval()
    model = model.cpu()
    return torch.quantization.convert(model)

quantized_model = converting(prepared_model)
torch.save(quantized_model, './weights/quantized_weight.pt')
```

</details>

<details><summary> <b>Second Method: QAT (Quantization Aware Training)</b> </summary>

```
step 1. Building float32 model or loading pre-trained weight (This step is the same as the step 1 of the above first method (PTQ))
```

```python
import torch.nn as nn
from torch.quantization import QuantStub, DeQuantStub

class Model(nn.Module):
    def __init__(self, model):
        self.model = model
        self.quant = QuantStub()
        self.dequant = DeQuantStub()

    def forward(self, x):
        x = self.quant(x)
        x = self.model(x)
        x = self.dequant(x)
        return x

model = Model()
```

```
step 2. Setting training mode for model and assigning to cpu device
```

```python
model.train()
model = model.cpu()
```

```
step 3. Fusing modules of model (This step is the same as the step 3 of the above first method (PTQ))
```

```python
import torch

def fuse_modules(model):
    model = model.cpu()

    modules = [
        ['conv1', 'bn1'],
    ]

    return torch.quantization.fuse_modules(model, modules)

fused_model = fuse_modules(model)
```

```
step 4. Setting qconfig and preparing quantization
```

```python
import torch

def prepare_qat(model, backend: str='x86'):
    model.train()
    model = model.cpu()
    model.qconfig = torch.quantization.get_default_qat_qconfig(backend)
    return torch.quantization.prepare_qat(model)

prepared_model = prepare_qat(fused_model)
```

```
step 5. Training model on GPU device (QAT step)
```

```python
training(model, train_loader, device=torch.device('cuda'))

```

```
step 6. Converting the weight from float32 to uint8 and saving the quantized weight
```

```python
import torch

def converting(model):
    model.eval()
    model = model.cpu()
    return torch.quantization.convert(model)

quantized_model = converting(model)
torch.save(quantized_model, './weights/quantized_weight.pt')
```

</details>

### References

- [PyTorch document](https://pytorch.org/docs/stable/quantization.html)
- [My Repository](https://github.com/Sangh0/Quantization)
