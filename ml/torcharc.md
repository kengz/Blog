---
description: 2020/06/20
---

# TorchArc

Building modular PyTorch models for my projects in the past years has prompted me to use a config-based approach to define model architecture. Over time I have iteratively refined the method, and recently I felt it has become sufficiently mature to be open sourced.

The project is known as [TorchArc](https://github.com/kengz/torcharc): Build PyTorch networks by specifying architectures. You can install it from pip:

```bash
pip install torcharc
```

My experience from building quite a lot of models for DL and RL \(which can have unconventional architecture\) has resulted in the following observations:

* most models are built with common components and hyperparameters - layers, width, activation, norm, dropout, init, etc. These can be specified via a config with the structure of a JSON/YAML.
* using config-based architecture frees us from frequent hard-code changes, while also immediately allow for hyperparameter optimization on the entire architecture. Yes, you can do NAS \(neural architecture search\) quite easily.
* sometimes we wish to compose models together, e.g. a hybrid network with a conv net and MLP for bi-modal inputs, joined together in the middle with another MLP, and split to multiple outputs for multi-modal controls.
* The composed networks is always a DAG. This means it can be specified via a JSON/YAML structure too \(this can be proven mathematically, but it's not why we're here\).

These essentially formed the design requirement for TorchArc. Additionally, I've also added a `carry_forward` method which accepts a TensorTuple input, forward-pass any tensors in it by name-matching, and carry any unused tensors in the output. This allows a multi-modal input to be carried and forward-passed all the way until the output.

Let's jump straight into TorchArc.

## Example Usage

Given just the architecture, `torcharc` can build generic DAG \(directed acyclic graph\) of nn modules, which consists of:

* single-input-output modules: `Conv1d, Conv2d, Conv3d, Linear, PTTSTransformer, TSTransformer` or any other valid nn.Module
* fork modules: `ReuseFork, SplitFork`
* merge modules: `ConcatMerge, FiLMMerge`

The custom modules are defined in [`torcharc/module`](https://github.com/kengz/torcharc/tree/master/torcharc/module), registered in [`torcharc/module_builder.py`](https://github.com/kengz/torcharc/blob/master/torcharc/module_builder.py).

The full examples of architecture references are in [`torcharc/arc_ref.py`](https://github.com/kengz/torcharc/blob/master/torcharc/arc_ref.py), and full functional examples are in [`test/module/`](https://github.com/kengz/torcharc/tree/master/test/module). Below we walk through some main examples.

### ConvNet

{% tabs %}
{% tab title="arc" %}
```python
import torcharc


arc = {
    'type': 'Conv2d',
    'in_shape': [3, 20, 20],
    'layers': [
        [16, 4, 2, 0, 1],
        [16, 4, 1, 0, 1]
    ],
    'batch_norm': True,
    'activation': 'ReLU',
    'dropout': 0.2,
    'init': 'kaiming_uniform_',
}
model = torcharc.build(arc)

batch_size = 16
x = torch.rand([batch_size, *arc['in_shape']])
y = model(x)
```
{% endtab %}

{% tab title="model" %}
```text
Sequential(
  (0): Conv2d(3, 16, kernel_size=(4, 4), stride=(2, 2))
  (1): BatchNorm2d(16, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
  (2): ReLU()
  (3): Dropout2d(p=0.2, inplace=False)
  (4): Conv2d(16, 16, kernel_size=(4, 4), stride=(1, 1))
  (5): BatchNorm2d(16, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
  (6): ReLU()
  (7): Dropout2d(p=0.2, inplace=False)
)
```
{% endtab %}
{% endtabs %}

### MLP

{% tabs %}
{% tab title="arc" %}
```python
arc = {
    'type': 'Linear',
    'in_features': 8,
    'layers': [64, 32],
    'batch_norm': True,
    'activation': 'ReLU',
    'dropout': 0.2,
    'init': {
        'type': 'normal_',
        'std': 0.01,
    },
}
model = torcharc.build(arc)

batch_size = 16
x = torch.rand([batch_size, arc['in_features']])
y = model(x)
```
{% endtab %}

{% tab title="model" %}
```text
Sequential(
  (0): Linear(in_features=8, out_features=64, bias=True)
  (1): BatchNorm1d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
  (2): ReLU()
  (3): Dropout(p=0.2, inplace=False)
  (4): Linear(in_features=64, out_features=32, bias=True)
  (5): BatchNorm1d(32, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
  (6): ReLU()
  (7): Dropout(p=0.2, inplace=False)
)
```
{% endtab %}
{% endtabs %}

### Time-Series Transformer

{% tabs %}
{% tab title="arc" %}
```python
arc = {
    'type': 'TSTransformer',
    'd_model': 64,
    'nhead': 8,
    'num_encoder_layers': 4,
    'num_decoder_layers': 4,
    'dropout': 0.2,
    'dim_feedforward': 2048,
    'activation': 'relu',
    'in_embedding': 'Linear',
    'pe': 'sinusoid',
    'attention_size': None,
    'in_channels': 1,
    'out_channels': 1,
    'q': 8,
    'v': 8,
    'chunk_mode': None,
}
model = torcharc.build(arc)

seq_len = 32
x = torch.rand([seq_len, arc['in_channels']])
```
{% endtab %}

{% tab title="model" %}
```text
TSTransformer(
  (in_embedding): Linear(in_features=1, out_features=64, bias=True)
  (pe): SinusoidPE(
    (dropout): Dropout(p=0.1, inplace=False)
  )
  (encoders): Sequential(
    (0): Encoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
    (1): Encoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
    (2): Encoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
    (3): Encoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
  )
  (decoders): ModuleList(
    (0): Decoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_encoderDecoderAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm3): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
    (1): Decoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_encoderDecoderAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm3): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
    (2): Decoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_encoderDecoderAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm3): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
    (3): Decoder(
      (_selfAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_encoderDecoderAttention): MultiHeadAttention(
        (_W_q): Linear(in_features=64, out_features=64, bias=True)
        (_W_k): Linear(in_features=64, out_features=64, bias=True)
        (_W_v): Linear(in_features=64, out_features=64, bias=True)
        (_W_o): Linear(in_features=64, out_features=64, bias=True)
      )
      (_feedForward): PositionwiseFeedForward(
        (_linear1): Linear(in_features=64, out_features=2048, bias=True)
        (_linear2): Linear(in_features=2048, out_features=64, bias=True)
      )
      (_layerNorm1): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm2): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_layerNorm3): LayerNorm((64,), eps=1e-05, elementwise_affine=True)
      (_dropout): Dropout(p=0.2, inplace=False)
    )
  )
  (out_linear): Linear(in_features=64, out_features=1, bias=True)
)
```
{% endtab %}
{% endtabs %}

### DAG: Hydra

Ultimately, we can build a generic DAG network using the modules linked by the fork and merge modules. The example below shows HydraNet - a network with multiple inputs and multiple outputs.

{% tabs %}
{% tab title="arc" %}
```python
arc = {
    'dag_in_shape': {'image': [3, 20, 20], 'vector': [8]},
    'image': {
        'type': 'Conv2d',
        'in_names': ['image'],
        'layers': [
            [16, 4, 2, 0, 1],
            [16, 4, 1, 0, 1]
        ],
        'batch_norm': True,
        'activation': 'ReLU',
        'dropout': 0.2,
        'init': 'kaiming_uniform_',
    },
    'merge': {
        'type': 'FiLMMerge',
        'in_names': ['image', 'vector'],
        'names': {'feature': 'image', 'conditioner': 'vector'},
    },
    'Flatten': {
        'type': 'Flatten'
    },
    'Linear': {
        'type': 'Linear',
        'layers': [64, 32],
        'batch_norm': True,
        'activation': 'ReLU',
        'dropout': 0.2,
        'init': 'kaiming_uniform_',
    },
    'out': {
        'type': 'Linear',
        'out_features': 8,
    },
    'fork': {
        'type': 'SplitFork',
        'shapes': {'mean': [4], 'std': [4]},
    }
}
model = torcharc.build(arc)

batch_size = 16
dag_in_shape = arc['dag_in_shape']
xs = {'image': torch.rand([batch_size, *dag_in_shape['image']]), 'vector': torch.rand([batch_size, *dag_in_shape['vector']])}
# returns dict of Tensors if output is multi-modal, Tensor otherwise
ys = model(xs)
```
{% endtab %}

{% tab title="model" %}
```text
DAGNet(
  (module_dict): ModuleDict(
    (image): Sequential(
      (0): Conv2d(3, 16, kernel_size=(4, 4), stride=(2, 2))
      (1): BatchNorm2d(16, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (2): ReLU()
      (3): Dropout2d(p=0.2, inplace=False)
      (4): Conv2d(16, 16, kernel_size=(4, 4), stride=(1, 1))
      (5): BatchNorm2d(16, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (6): ReLU()
      (7): Dropout2d(p=0.2, inplace=False)
    )
    (merge): FiLMMerge(
      (conditioner_scale): Linear(in_features=8, out_features=16, bias=True)
      (conditioner_shift): Linear(in_features=8, out_features=16, bias=True)
    )
    (Flatten): Flatten()
    (Linear): Sequential(
      (0): Linear(in_features=576, out_features=64, bias=True)
      (1): BatchNorm1d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (2): ReLU()
      (3): Dropout(p=0.2, inplace=False)
      (4): Linear(in_features=64, out_features=32, bias=True)
      (5): BatchNorm1d(32, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (6): ReLU()
      (7): Dropout(p=0.2, inplace=False)
    )
    (out): Linear(in_features=32, out_features=8, bias=True)
    (fork): SplitFork()
  )
)
```
{% endtab %}
{% endtabs %}

The DAG module accepts a `dict` \(example below\) as input, and the module selects its input by matching its own name in the arc and the `in_name`, then carry forward the output together with any unconsumed inputs.

For example, the input `xs` with keys `image, vector` passes through the first `image` module, and the output becomes `{'image': image_module(xs.image), 'vector': xs.vector}`. This is then passed through the remainder of the modules in the arc as declared.



