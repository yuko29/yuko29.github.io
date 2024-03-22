---
title: "Multi-head Attention"
date: 2024-03-22T13:15:51+08:00
categories: ['Deep Learning']
tags: []
draft: False
---

詳解 Multi-head Attention 實作。  

<!--more-->

## Attention

Attention 的算法公式列在下方：

$$A(Q, K, V) = \text{softmax} ( \frac{QK^T}{\sqrt{d_k}})V$$

其中，Q, K, V 矩陣是 input tensor 經過分別三個 linear projection 得到。  
上述的 attention 是 single head 的，只有計算一組 Q, K, V。所謂 Multi-head Attention，則是希望同時有多組的 Q, K, V，讓模型可以學習到不同的 feature。

## Multi-head Attention

先上程式碼，以 Huggingface 的 LlamaAttention 實作舉例，只留下核心算法的部份：  

```python
class LlamaAttention(nn.Module):
    """Multi-headed attention from 'Attention Is All You Need' paper"""

    def __init__(self, config: LlamaConfig, ratio=[1,1,1,1]):
        super().__init__()
        self.config = config
        self.hidden_size = config.hidden_size
        self.num_heads = config.num_attention_heads
        self.head_dim = self.hidden_size // self.num_heads
        self.max_position_embeddings = config.max_position_embeddings
        self.ratio = ratio # 1 means no truncate, just keep normal attn

        if (self.head_dim * self.num_heads) != self.hidden_size:
            raise ValueError(
                f"hidden_size must be divisible by num_heads (got `hidden_size`: {self.hidden_size}"
                f" and `num_heads`: {self.num_heads})."
            )
        
        self.q_proj = nn.Linear(self.hidden_size, self.num_heads * self.head_dim, bias=False)
        self.k_proj = nn.Linear(self.hidden_size, self.num_heads * self.head_dim, bias=False)
        self.v_proj = nn.Linear(self.hidden_size, self.num_heads * self.head_dim, bias=False)
        self.o_proj = nn.Linear(self.num_heads * self.head_dim, self.hidden_size, bias=False)
        self.rotary_emb = LlamaRotaryEmbedding(self.head_dim, max_position_embeddings=self.max_position_embeddings)

    def forward(
            self,
            hidden_states: torch.Tensor,
            attention_mask: Optional[torch.Tensor] = None,
            position_ids: Optional[torch.LongTensor] = None,
            past_key_value: Optional[Tuple[torch.Tensor]] = None,
            output_attentions: bool = True,
            use_cache: bool = False,
        ) -> Tuple[torch.Tensor, Optional[torch.Tensor], Optional[Tuple[torch.Tensor]]]:
            bsz, q_len, _ = hidden_states.size()
        
        query_states = self.q_proj(hidden_states).view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)
        key_states = self.k_proj(hidden_states).view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)
        value_states = self.v_proj(hidden_states).view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)

        attn_weights = torch.matmul(query_states, key_states.transpose(2, 3)) / math.sqrt(self.head_dim)
        attn_weights = nn.functional.softmax(attn_weights, dim=-1, dtype=torch.float32).to(query_states.dtype)
        attn_output = torch.matmul(attn_weights, value_states)
        attn_output = attn_output.transpose(1, 2) # group by token
        attn_output = attn_output.reshape(bsz, q_len, -1)
```

以下分段解析 forward 的部份。

### Step 1

在實作方法中，Q, K, V 的 linear projection 仍然只用各一個 linear projection 完成 (`q_proj`, `k_proj`, `v_proj`)。  

```python
query_states = self.q_proj(hidden_states).view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)
key_states = self.k_proj(hidden_states).view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)
value_states = self.v_proj(hidden_states).view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)
```

{{< img src="images/inputq.svg">}}

{{< img src="images/qko.svg">}}

### Step 2

{{% admonition info "info" %}}
先提一下整個 MHA 實作的中心思想：  
把原本 hidden dim 平均拆成 num_heads 塊，這些子塊代表不同的 head，各自獨立計算 attention，最後再將結果拼回一起。
{{% /admonition %}}

此步驟的目的是要將 Q, K, V 沿著 hidden dim 維度平均拆成 num_heads 份。      

對 Q, K, V 做 reshape，把原本 token 的 hidden dim 切成 num_heads x head_dim，如下方中間的圖。每個 token 的 hidden dim 被分成 num_heads 組了。
再來，做 transpose 把 num_heads 和 seq_len 這兩個維度對調，把同的 head 部份拼在一起。

{{< img src="images/reshape_transpose.svg">}}

到這邊可以發現，這兩個步驟其實等同於把原本的矩陣依照 hidden dim 維度分割成 num_heads 塊。  
因此我們得到了 Multi-head Attention 需要的多組 Q, K, V (即 \(q_i, k_i, v_i, i=0, 1, \dots \text{,num_heads-1}\))。

{{< img src="images/qko_transpose.svg">}}

好像有點難想像？

舉間單例子：seq_len = 4, hidden_dim = 6, num_heads = 2, head_dim = 3

```
>>> a = torch.rand(4,6)
>>> a
tensor([[0.7924, 0.4975, 0.5119, 0.4034, 0.6843, 0.3314],
        [0.9629, 0.7819, 0.5565, 0.5319, 0.2773, 0.2841],
        [0.3263, 0.7731, 0.5472, 0.6618, 0.3387, 0.6278],
        [0.7786, 0.0196, 0.0878, 0.0646, 0.6827, 0.6362]])

>>> a.view(4,2,3)
tensor([[[0.7924, 0.4975, 0.5119],
         [0.4034, 0.6843, 0.3314]],

        [[0.9629, 0.7819, 0.5565],
         [0.5319, 0.2773, 0.2841]],

        [[0.3263, 0.7731, 0.5472],
         [0.6618, 0.3387, 0.6278]],

        [[0.7786, 0.0196, 0.0878],
         [0.0646, 0.6827, 0.6362]]])

>>> a.view(4,2,3).transpose(0,1)
tensor([[[0.7924, 0.4975, 0.5119],
         [0.9629, 0.7819, 0.5565],
         [0.3263, 0.7731, 0.5472],
         [0.7786, 0.0196, 0.0878]],

          
        [[0.4034, 0.6843, 0.3314],
         [0.5319, 0.2773, 0.2841],
         [0.6618, 0.3387, 0.6278],
         [0.0646, 0.6827, 0.6362]]])
```

很巧妙對吧。看到這裡我不禁驚嘆出聲。

### Step 3

```python
attn_weights = torch.matmul(query_states, key_states.transpose(2, 3)) / math.sqrt(self.head_dim)
attn_weights = nn.functional.softmax(attn_weights, dim=-1, dtype=torch.float32).to(query_states.dtype)
attn_output = torch.matmul(attn_weights, value_states)
```

對各組 \(q_i, k_i, v_i\) 個別計算 attention，得到單個 head 的 attention output \(s_i\)，這邊利用高維矩陣相乘實作。  
高維矩陣相乘，其實就是對於高維矩陣中的每個二維矩陣做矩陣乘法。剛剛處理完的 Q, K, V 裡含有 num_heads 個二維矩陣，因此對對應的 \(q_i,k_i^T,v_i\) 計算 attention 這件事在編寫程式上就可以用兩次矩陣乘法完成。

{{< img src="images/mha.svg">}}

### Step 4

計算完所有 heads 的 attention 得到 num_heads 個 \(s_i\) 後，實行 [Step 2]({{<ref "#step-2" >}}) 的逆操作。  
先將 seq_len 和 num_heads 維度對調 (transpose) ，相當於把屬於個別 token 的 hidden dim 子塊拼起來，再把矩陣 reshape 成 (seq_len, hidden dim)。  
至此，原本各自計算的 \(s_i\) 被合回一個二維矩陣，完成 Multi-head Attention 的計算。

{{< img src="images/transpose_back.svg">}}

### Step 5

在 Multi-head Attention 運算最後，常會把結果再做一次 linear projection，稱為 output projection。得到的結果為最終 Multi-head Attention 的 output。

{{< img src="images/output.svg">}}

> **後記**  
>   
> 看了很多 Multihead attention 的解說但一直沒能懂，那些圖完全沒能輔助我解讀計算過程。徹底把程式碼拆出來看後我才徹徹底底的理解 MHA 到底是個什麼樣的機制，並把理論很緊密和實作的關聯起來。因此決定寫一篇 blog 把我的理解過程寫下來備忘，並順手畫一些圖輔助理解。

{{< bmc-button slug="yuko29" >}}