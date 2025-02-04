# English to Hungarian Translation using Transformer

This project utilizes a Transformer model for English to Hungarian translation. The model is trained on the "opus_books" dataset from the Hugging Face's dataset hub.

## Requirements

- Python 3.11 or later
- PyTorch 2.0.1
- Transformers 4.30.0 or later
- pandas 1.5.3 or later
- numpy 1.23.5 or later
- scikit-learn 1.2.2 or later

## Installation

Clone the repository and install the required packages:

```bash
git clone https://github.com/ThanhBuiKhacNhat/Transformer-translation-task.git
cd Transformer-translation-task
pip install -q git+https://github.com/huggingface/transformers.git
pip install datasets
pip install -r requirements.txt
```



# Table of Contents
- [Table of Contents](#table-of-contents)
- [Transformer - Attention is all you need - Pytorch Implementation](#transformer---attention-is-all-you-need---pytorch-implementation)
- [Models](#models)
  - [Positional Encoding](#positional-encoding)
  - [Scaled Dot-Product Attention](#scaled-dot-product-attention)
  - [Multi-Head Attention](#multi-head-attention)
  - [Encoder](#encoder)
    - [Encoder Layer](#encoder-layer)
    - [Encoder](#encoder-1)
  - [Transformer](#transformer)
- [Training](#training)
  - [Download the dataset](#download-the-dataset)
  - [Train the model](#train-the-model)
- [Evaluation](#evaluation)
- [Inference](#inference)
- [References](#references)

# Transformer - Attention is all you need - Pytorch Implementation

<p align="center">
<img src="https://miro.medium.com/max/1400/1*BHzGVskWGS_3jEcYYi6miQ.png" width="700">
</p>


The project support training and translation with trained model now.

If there is any suggestion or error, feel free to fire an issue to let me know. :)

The directory structure of this project is shown below:
```bash
- `dataset.py`: Contains the `TranslationDataset` class and `load_datasets` function for preparing the data.
- `models.py`: Contains the `TransformerTranslator` model definition.
- `train.py`: Contains the training loop logic.
- `evaluate.py`: Contains the evaluation logic.
- `translate.py`: Contains a function to translate sentences using the trained model.
- `main.py`: The main script to tie everything together and run the training and evaluation.
- `README.md`: Project description and instructions.
```

---
# Models

## Positional Encoding
The positional encodings have the same dimension d_model as the embeddings, so that the two can be summed.
```python
class PositionalEncoder(nn.Module):
    def __init__(self, d_model, max_seq_len=512):
        super(PositionalEncoder, self).__init__()
        self.d_model = d_model
        self.max_seq_len = max_seq_len
        self.dropout = nn.Dropout(p=0.1)
        
        # Compute positional encodings
        position = torch.arange(0, max_seq_len, dtype=torch.float).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-torch.log(torch.tensor(10000.0)) / d_model))
        self.positional_encoding = torch.zeros(max_seq_len, d_model)
        self.positional_encoding[:, 0::2] = torch.sin(position * div_term)
        self.positional_encoding[:, 1::2] = torch.cos(position * div_term)
        self.positional_encoding = self.positional_encoding.unsqueeze(0)
    
    def forward(self, x):
        # Add positional encodings to input embeddings
        x = x + self.positional_encoding[:, :x.size(1)].to(x.device)
        return self.dropout(x)
```

## Multi-Head Attention



```python
# Multi-head attention layer
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super(MultiHeadAttention, self).__init__()
        assert d_model % num_heads == 0, "d_model must be divisible by num_heads"
        self.d_model = d_model
        self.num_heads = num_heads
        self.depth = d_model // num_heads
        
        self.query = nn.Linear(d_model, d_model)
        self.key = nn.Linear(d_model, d_model)
        self.value = nn.Linear(d_model, d_model)
        self.fc = nn.Linear(d_model, d_model)
    
    def split_heads(self, x, batch_size):
        x = x.view(batch_size, -1, self.num_heads, self.depth)
        return x.permute(0, 2, 1, 3)
    
    def forward(self, query, key, value, mask):
        batch_size = query.size(0)
        
        query = self.split_heads(self.query(query), batch_size)
        key = self.split_heads(self.key(key), batch_size)
        value = self.split_heads(self.value(value), batch_size)
        
        scaled_attention_logits = torch.matmul(query, key.transpose(-2, -1)) / torch.sqrt(torch.tensor(self.depth, dtype=torch.float32))
        
        if mask is not None:
            mask = mask.unsqueeze(1).unsqueeze(2)
            scaled_attention_logits += mask * -1e9
        
        attention_weights = F.softmax(scaled_attention_logits, dim=-1)
        
        output = torch.matmul(attention_weights, value)
        output = output.permute(0, 2, 1, 3).contiguous().view(batch_size, -1, self.d_model)
        
        return self.fc(output)
```
## Encoder
### Encoder Layer
```python
# Transformer encoder layer
class EncoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, hidden_dim, dropout=0.1):
        super(EncoderLayer, self).__init__()
        self.mha = MultiHeadAttention(d_model, num_heads)
        self.dropout1 = nn.Dropout(dropout)
        self.norm1 = NormLayer(d_model)
        
        self.ffn = nn.Sequential(
            nn.Linear(d_model, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, d_model)
        )
        self.dropout2 = nn.Dropout(dropout)
        self.norm2 = NormLayer(d_model)
    
    def forward(self, x, mask):
        attention_output = self.mha(x, x, x, mask)
        attention_output = self.dropout1(attention_output)
        out1 = self.norm1(x + attention_output)
        
        ffn_output = self.ffn(out1)
        ffn_output = self.dropout2(ffn_output)
        out2 = self.norm2(out1 + ffn_output)
        
        return out2
```

<!-- <p align="center">
<img src="https://www.factored.ai/wp-content/uploads/2021/09/image2-580x1024.png" width="350">
</p> -->

<figure>
<p align="center">
<img src="https://www.factored.ai/wp-content/uploads/2021/09/image2-580x1024.png" width="350">
</p>
<figcaption>
Encoder: The encoder is composed of a stack of <b>N = 6</b> identical layers. Each layer has two sub-layers. The first is a multi-head self-attention mechanism, and the second is a simple, positionwise fully connected feed-forward network. We employ a residual connection around each of the two sub-layers, followed by layer normalization. That is, the output of each sub-layer is LayerNorm(x + Sublayer(x)), where Sublayer(x) is the function implemented by the sub-layer itself. To facilitate these residual connections, all sub-layers in the model, as well as the embedding layers, produce outputs of dimension.
</figcaption>
</figure>

### Encoder
```python
# Encoder transformer
class Encoder(nn.Module):
    def __init__(self, num_layers, d_model, num_heads, hidden_dim, dropout=0.1):
        super(Encoder, self).__init__()
        self.num_layers = num_layers
        self.layers = nn.ModuleList([EncoderLayer(d_model, num_heads, hidden_dim, dropout) for _ in range(num_layers)])
    
    def forward(self, x, mask):
        for layer in self.layers:
            x = layer(x, mask)
        return x
```



## Transformer
```python
# Transformers
class TransformerTranslator(nn.Module):
    def __init__(self, num_layers, d_model, num_heads, hidden_dim, input_vocab_size, target_vocab_size, max_seq_len, dropout=0.1, learning_rate=1e-2, batch_size=128):
        super(TransformerTranslator, self).__init__()
        self.embedding = nn.Embedding(input_vocab_size, d_model)
        self.positional_encoder = PositionalEncoder(d_model, max_seq_len)
        self.encoder = Encoder(num_layers, d_model, num_heads, hidden_dim, dropout)
        self.fc = nn.Linear(d_model, target_vocab_size)
        self.learning_rate = learning_rate
        self.batch_size = batch_size
    
    def forward(self, input_ids, mask):
        x = self.embedding(input_ids)
        x = self.positional_encoder(x)
        x = self.encoder(x, mask)
        x = self.fc(x)
        return x
```

---
# Usage
## Download the dataset
```bash
pip install -q git+https://github.com/huggingface/transformers.git
pip install datasets
```
## Train the model
```bash
python train.py
```
Parameter settings:
- batch size : 128
- n_epochs : 20
- learning_rate : 1e-3
- num_heads : 8
- hidden_dim : 512
- d_model : 256
- num_layers : 6
- max_seq_len : 128
- dropout : 0.1

<p align="center">
<img src="plot.png" width="700">


# Evaluation
```bash
python evaluation.py
```
# Ineferences
See the file translate.py. If you don't have the resources to train the model, you can download my pre-trained model to use. (transformer_model.pth)
