---
layout: post
title:  "ChimeraNet+x-vectorでデータを集めたい その2"
date:   2020-04-13 06:00:00 +0900
---

今日は実験の内容について説明します。

### 実験の概要

#### 実験の目的
実験の目的は`ChimeraNet`による信号分離の結果が`x-vector`の話者推定に適しているかを調査することです。

#### 実験の主な内容(変更の可能性あり)

1. 信号分離前のデータを用いた話者推定の性能
2. 信号分離後のデータを用いた話者推定の性能
3. サンプル数の増加に対する性能変化

### 実験パラメータの説明

この節では、学習データを取り出すために必要な実験パラメータを説明します。
この実験パラメータをもとに学習データを取り出します。
実験パラメータは以下の組合せをもとに生成しました。

{% highlight python linenos %}
for n_trials, spk, (infer_type, quality_list), n_samples in\
product(
    range(20),
    ('Serval', 'Kaban'),
    (
        ('raw', 'AB'),
        ('raw', 'ABC'),
        ('embd', 'ABCD'),
        ('mask', 'ABCD')
    ),
    (520, 650, 880, 1000),
):
    # 上記の設定で実験ディレクトリを作成（後述）
    pass
{% endhighlight %}

各実験パラメータの内容は以下です。

| 名称           | 説明                 |
| :---           | :---                 |
| `n_trials`     | 試行回数(使用しない) |
| `spk`          | 話者名               |
| `infer_type`   | 信号分離アルゴリズム(後述) |
| `quality_list` | 使用する音声の品質(後述) |
| `n_samples`    | 使用するサンプルの数 |

`infer_type`の値の説明は以下です。

| 名称     | 説明                                       |
| :----  | :----                                      |
| `embd` | DeepClusteringによる埋め込み表現からの推定 |
| `mask` | softmaxによる推定                          |
| `raw`  | 信号分離なし                               |

`quality_list`の値の説明は以下です。

| 名称    | 説明                  |
| :---  | :---                  |
| `ABC` | 品質`A`,`B`,`C`を使用 |
| `D`   | 品質`D`を使用         |

### 実験ディレクトリを作成

実験ディレクトリは以下のKaldiデータディレクトリを含む
* `train`: PLDAモデルの訓練データ
* `test_X_Y`: テストデータ、`X`は信号分離アルゴリズム、`Y`は元音声の品質を表す

`X`および`Y`はそれぞれ学習データの`infer_type`および`quality_list`のものです。
そのため、一つの実験パラメータに対応するKaldiデータディレクトリは以下の7つです。

```
data/trial_########/
├── test_embd_ABC
├── test_embd_D
├── test_mask_ABC
├── test_mask_D
├── test_raw_ABC
├── test_raw_D
└── train
```


