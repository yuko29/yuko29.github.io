---
title: "LLM Evaluation"
date: 2024-02-03T02:16:38+08:00
categories: ['Deep Learning']
tags: ['LLM']
draft: false
---

如何評估 LLM 的能力？來看看學術界目前如何實作。  

<!--more-->

{{% admonition abstract 目的 %}}
主要以學術研究的角度切入討論。

- 如何評估 LLM 的能力
- 了解 LLM 如何生成文本
- 了解如何解讀技術報告裡的數據

{{% /admonition %}}

## Intro

LLM 的能力評估方式到目前在學界其實還是一個 open question，並沒有一個絕對的指標能夠完美的衡量一個 LLM 的能力。這個「能力」到底如何定義？這個能力終究是來自於我們希望 LLM 能夠做到的事，從我們會問 ChatGPT 的內容能夠知道我們關心哪些能力大概有哪些：

- 做學科題目
- 寫 code
- 產生各類文案 

因此不論是在 GPT 或是 LLaMA 的技術報告內，都是列出模型在多個不同面向的任務（task）上的準確度如何，來給出一個量化的數據呈現模型的能力。必須要注意，在解讀這些數字的時候要切記一點：任務的準確度只能反應模型在這項任務範疇內的表現。由此也可以理解為什麼報告內會列這麼多任務，就是為了要呈現出這個 LLM 它在各個面向表現的都很好。

但反過來想，是不是所有任務都要全部都測過一次，才可以有信心的說明這個模型很厲害呢？   
聽起來很合理，但是這些任務五花八門數量又很多，不太可能全部都測過，除非時間跟資源太多。因此，大家會去測幾個公認比較具有代表性的任務，來和別人比較自己的 LLM 能力。例如 MMLU，HellaSwag 等等，相當於是讓 LLM 去做各式各樣的題目來評估模型的能力。

所以，以目前的現有的方法來說，如果要量化評估 pretrained LLM 的能力的話，必須得透過任務這個媒介。選定一些任務作為評估能力的標準，從各方面衡量模型的表現，也才能夠在相同的基準點上公平地與其他 LLM 模型比較。

## LM Evaluation Harness

目前學術界論文中，大多是使用
[EleutherAI](https://www.eleuther.ai/) 所開發的 [LM Evaluation Harness](https://github.com/EleutherAI/lm-evaluation-harness) 這個工具來做 LLM 評測。它集成了許多常見的任務，可以透過 script 設定一次測試多個 task，十分方便。

## The way to LLM evaluation

在下去做 LLM 評測之前，有些重要的必備知識得先了解。

### 1. Zero-shot & Few-shot

[Few-shot prompting](https://promptingguide.azurewebsites.net/techniques/fewshot) 是來自 In-Context Learning 這個概念，表面上似乎是 LLM 模型能讀懂上下文做出更好的回答。具體來說，就是能夠藉由 prompting 限縮 LLM 的回答範圍，讓 LLM 能夠回答出我們期望的答案。因此，研究者們發現，如果在想要給 LLM 做的題目前先給 LLM 幾個題目和回答範例，有助於 LLM 回答的正確性。

在 MMLU task 的評測中，目前習慣會用 5-shot 設定，也就是將真正要問的題目接在五個範例後面，一起作為 input 輸入 LLM，具體方式如下：

```
        <input>                  # train
        A. <reference>
        B. <reference>
        C. <reference>
        D. <reference>
        Answer: <A/B/C/D>

        x N (N-shot)

        <input>                  # test
        A. <reference1>
        B. <reference2>
        C. <reference3>
        D. <reference4>
        Answer:
```

1-shot 實例：
```

        The pleura
        A. have no sensory innervation.
        B. are separated by a 2 mm space.
        C. extend into the neck.
        D. are composed of respiratory epithelium.
        Answer: C

        Which of the following terms describes the body's ability to maintain its normal state?
        A. Anabolism
        B. Catabolism
        C. Tolerance
        D. Homeostasis
        Answer:

    Target: D
```

而所謂的 zero-shot evaluation，就是指不給任何範例，讓 LLM 直接作答。這是最嚴苛的設定，也是目前學術論文中，普遍用來評測一個 pretrained LLM 的方式。

{{% admonition info "few-shot 歧義" %}}
這裡的 few-shot prompting 與 few-shot fine-tuning 是不同的，前者只靠 prompting 不需要訓練模型，後者則需要，並且與 
few-shot learning （from meta learning）是完全不同的兩個概念。
{{% /admonition %}}

### 2. Top-k & Top-p

LLM 會給出下一個詞的預測，也就是各個詞的機率分佈，要怎麼根據這些機率選擇下一個詞，稱為 Decode strategy。直接選擇機率最高的詞不一定是最好的（Greedy Decoding），詳細可以參考這篇[文章](https://www.zhihu.com/tardis/zm/art/647813179?source_id=1003)。

- Top-k sampling: 只考慮前 k 個機率最大的詞進行抽樣
- Top-p sampling: 只考慮前幾個加起來機率是 p 的詞，從裡面進行抽樣

由此可以增加生成文字的隨機性，這也解釋了為什麼我們在問 ChatGPT 的時候它每次回答都會不太一樣。   
如果 Top-k 和 Top-p 都啟用，則通常會設定 Top-p 在 Top-k 之後起作用。

### 3. Temperature

LLM 的超參數 T，用於控制生成文本的隨機性，也是一種 Decode strategy。

LLM 輸出是詞彙表中每個 token 的機率分佈，由 softmax 計算產生此機率分佈 P，也就是對 logits 做 softmax：

(logits: 如今通常是指模型最後一層的輸出，這個說法有些歧義，詳見[此處](https://www.zhihu.com/question/60751553))


$$
p \left( x_i \right) = \frac{e^{x_i}}{\Sigma^V_{j=1}e^{x_j}}
$$


Temperature 參數（T）用於在 softmax 時調整 logits：


$$
p \left( x_i \right) = \frac{e^{\frac{x_i}{T}}}{\Sigma^V_{j=1}e^{\frac{x_j}{T}}}
$$


通常設置 0.1 到 1.0 之間，T 設越小，機率差異會被放大，結果越確定；T 設越大，分佈越平均，結果越隨機。在使用上，可以依據需要調整 temperature。

不過，在學術論文中，使用 LM Evaluation Harness 做 zero-shot evalution 的部份，並沒有看到過有作者特別說明 temperature 和 top-k & top-p 設定。如果要確保數據能重現的話，應該是沒有設 temperature 的。也有可能是，做題的時候不要有隨機性比較好。

LM Evaluation Harness 在沒有指定 temperature 的情況下預設是不會使用這個參數的 ([source code](https://github.com/EleutherAI/lm-evaluation-harness/blob/994bdb3fd3780110e45f479f5f409cae05e089bd/lm_eval/models/huggingface.py#L717))。


題外話，LLaMA-2 模型發布的時候有給 `generation_config.json`：

```
{
  "bos_token_id": 1,
  "do_sample": true,
  "eos_token_id": 2,
  "pad_token_id": 0,
  "temperature": 0.6,
  "max_length": 4096,
  "top_p": 0.9,
  "transformers_version": "4.31.0.dev0"
}
```

沒特別設定的話，用 huggingface transformers 加載模型就會吃這個設定。

## Metric

LLM 領域中，以壓縮為目的的論文，通常在 LLM 的能力評估上會比兩種指標：perplexity 和 task accuracy。

### 1. Perplexity

[Perplexity](https://docs.kolena.io/metrics/perplexity/)，簡稱 PPL，翻譯成困惑度，是衡量模型在 Language Modeling 任務能力的指標。PPL 反應的是模型預測字的能力。在訓練一個語言模型的時候，我們期待 LLM 能夠產生文字合理的句子，於是希望它在做 text generation 時，能夠接出與 training sample 一樣的句子，而 PPL 就是在衡量模型預測結果的機率組合和原句子的差異性。

Perplexity 的數學定義如下：
> Perplexity is defined as the exponential of the negative log-likelihood of a sequence.
> $$ \text{perplexity} = exp \left( -\frac{1}{t} \\ \sum\limits_1^t \\ \text{log} \\ p_{ \theta } \left( x_i | x_{context} \right) \right)$$

講白話一點就是 LLM 話說的好不好，loss 越小，PPL 值就越小，表示模型能力越好。

這裡有[完整的 code 範例](https://www.educative.io/answers/what-is-perplexity-in-nlp)，教導如何使用 huggingface transformer 算 PPL。

{{% admonition warning "warning" %}}
Text preprocessing 和 sequence length 都會影響 PPL 的值，比較時須在同樣的設定下測量。 
{{% /admonition %}}

 

另外，對不同的 dataset 量測 PPL，只是反應模型在該 dataset 上的預測程度，因此，在某個 dataset 上 PPL 大不能直接推論該 LLM 綜合能力就差。

### 2. Accuracy

Accuracy，簡稱 acc，在常見的 zero-shot task 大多是以題目形式為主，所以衡量方法就是看答對率。  

下面是 LM Evaluation Harness 跑出來的結果範例。

Zero-shot tasks:

```
|    Tasks    |Version|Filter|n-shot| Metric |Value |   |Stderr|                                                                                     
|-------------|-------|------|-----:|--------|-----:|---|-----:|                                                                                     
|arc_challenge|Yaml   |none  |     0|acc     |0.3217|±  |0.0137|                                                                                     
|             |       |none  |     0|acc_norm|0.3481|±  |0.0139|                                                                                     
|arc_easy     |Yaml   |none  |     0|acc     |0.6700|±  |0.0096|                                                                                     
|             |       |none  |     0|acc_norm|0.6077|±  |0.0100|                                                                                     
|boolq        |Yaml   |none  |     0|acc     |0.6835|±  |0.0081|                                                                                     
|hellaswag    |Yaml   |none  |     0|acc     |0.4703|±  |0.0050|                                                                                     
|             |       |none  |     0|acc_norm|0.6228|±  |0.0048|                                                                                     
|openbookqa   |Yaml   |none  |     0|acc     |0.2760|±  |0.0200|                                                                                     
|             |       |none  |     0|acc_norm|0.4160|±  |0.0221|                                                                                     
|piqa         |Yaml   |none  |     0|acc     |0.7285|±  |0.0104|                                                                                     
|             |       |none  |     0|acc_norm|0.7372|±  |0.0103|                                                                                     
|winogrande   |Yaml   |none  |     0|acc     |0.6251|±  |0.0136|
```

5-shot MMLU task:

```
|      Groups      |Version|Filter|n-shot|Metric|Value |   |Stderr|
|------------------|-------|------|-----:|------|-----:|---|-----:|
|mmlu              |N/A    |none  |     0|acc   |0.2745|±  |0.0509|
| - humanities     |N/A    |none  |     5|acc   |0.2482|±  |0.0326|
| - other          |N/A    |none  |     5|acc   |0.3025|±  |0.0502|
| - social_sciences|N/A    |none  |     5|acc   |0.2746|±  |0.0524|
| - stem           |N/A    |none  |     5|acc   |0.2858|±  |0.0638|
```

### 3. Accuracy Norm

在上面的結果中會看到，有些 task 會回報 `acc_norm`，這是什麼意思呢？

EleutherAI 的一篇 [blog post](https://blog.eleuther.ai/multiple-choice-normalization/) 裡有給出說明：

> Byte-length normalized: The score of continuation i is determined using \(\sum\limits_{j=m}^{n_{i−1}} \text{log ⁡P}(x_j|x_{0:j})/\sum \limits_{j=m}^{n_{i−1}} \text{L}_{x_j}\), where \(\text{L}_{x_j}\)  is the number of bytes represented by the token \(x_j\). This approach attempts to normalize for length by computing average log probability per character, which ensures that it is tokenization agnostic. This approach is also used by eval harness in all multiple choice tasks and presented as `acc_norm`.

也就是對機率分佈 P 除以字串長度，以去除 tokenization 的影響。LM Evaluation Harness 在 multiple choice task 類別的任務裡有應用這個方法，對選項 token 長度做 normalization，其結果就是 `acc_norm`。


```python
def process_results(self, doc: dict, results: List[Tuple[float, bool]]) -> dict:
    results = [
        res[0] for res in results
    ]
    gold = doc["gold"]

    acc = 1.0 if np.argmax(results) == gold else 0.0
    completion_len = np.array([float(len(i)) for i in doc["choices"]])
    acc_norm = 1.0 if np.argmax(results / completion_len) == gold else 0.0

    return {
        "acc": acc,
        "acc_norm": acc_norm,
    }
```

在某些任務上，`acc_norm` 的值會比較高，表示 LLM 在這個任務比較會受到選項長度影響。不過在目前我所看到的論文中，大部分的作者都是呈現 `acc`，而不是 `acc_norm`。不會在 LLM 的技術報告上有些數字看起來是 `acc_norm`，比較時需要留意。


## Ending

LLM 本身如同一個不知道有幾面的多面體，我認為要完備的評估它是不可能的事，只能用像這種多面推敲的方式來探索這個模型的能力。即使是這樣，現階段這些手段確實也能夠一定程度反應模型能力，但還有改善的空間。

本篇文章只有和涵蓋部份的 LLM evaluation 方法，其他像是 ROUGE 之類的 NLG 任務指標就沒講了，之後有機會再談吧。

> 一些研究過程的 murmur。   
2023 年，從 ChatGPT 釋出到 LLaMA 2 開源，各方團隊都在煉自家的 LLM，arXiv 上相關的論文也如噴湧而出。近期的研究工作也跨入了 LLM 領域，本身在 NLP 領域是新手，在紛亂的論文中蒐集資料實在吃盡苦頭。網路上雖然有些文章分享了 LLM 評估方法但都不夠詳細，包含已發表的論文也是，從規劃研究開始在評估方法方面就遇到巨大困擾，花了很多時間才找到對的 benchmark 和評估方法，跑出論文上的數字。數據是跑出來了，但有些疑惑漸漸浮現：
> * 這些 benchmark 到底能反應 LLM 多少程度的能力？
> * 對於那些沒有列出來的任務，其表現又如何呢？   
> 
> 為了解決這些疑惑，決定好好把這些任務指標和評測方法弄清楚。這使得我在後續理解論文數據的過程順暢很多。


## Reference

- [Temperature — LLMs](https://medium.com/@amansinghalml_33304/temperature-llms-b41d75870510)
- [语言模型中的常用评估指标](http://giantpandacv.com/project/PyTorch/%E5%A4%A7%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E4%B8%AD%E7%9A%84%E5%B8%B8%E7%94%A8%E8%AF%84%E4%BC%B0%E6%8C%87%E6%A0%87/)
- [lm-evaluation-harnessの評価指標まとめ](https://zenn.dev/hijikix/articles/28a134e41d7e75)


{{< bmc-button slug="yuko29" >}}