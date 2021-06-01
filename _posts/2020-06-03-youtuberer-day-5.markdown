---
layout: post
title:  "あつまれ YouTuberのデータ その5"
date:   2020-06-03 20:00:00 +0900
published: false
---

[前回]({% post_url 2020-05-05-youtuberer-day-4 %})
[次回]({% post_url 2021-03-28-youtuberer-day-6 %})

前回までの手順で収集した動画リストと[`pafy`](https://pypi.org/project/pafy/)と[`syncnet_python`](https://github.com/joonson/syncnet_python)を使って音声データを収集します。

### 目次

1. 大まかな流れ
2. syncnet_pythonの実行
4. Confidenceのヒストグラム
5. 次回予告

### 1 大まかな流れ

<ol start="0">
<li>分析対象の動画リストを用意(前回まで参照)</li>
<li>`pafy`でダウンロード</li>
<li>`syncnet_python`で分析</li>
<li>結果をコピー</li>
</ol>

この中の'`synenct_python`で分析'については若干説明します。

### 2 synenct_pythonの実行

`synenct_python`公式READMEによると、以下3つのスクリプトで処理を行います。

1. `run_pipeline.py`
2. `run_syncnet.py`
3. `run_visualise.py`

このうち、`run_visualise.py`は元の動画の顔の周りに枠を描画するだけのスクリプトなので、このプロジェクトでは不採用とします。

まずは学習済みネットワークをダウンロードするスクリプト`download_model.sh`を実行します。

次に`run_pipeline.py`を実行します。
`run_pipeline.py`は[`S3FD`](https://arxiv.org/abs/1708.05237)という方法で元の動画
から顔が映っている部分だけを抜き取った動画を保存します。
同時に、抜き取ったフレームに関する情報(位置、フレーム区間など)も保存します。

保存場所は`--data_dir`オプションで渡したディレクトリのサブディレクトリ`pycrop`(顔動画)と`pywork`(付随情報)です。より具体的には以下のファイルに保存されます。

```
--data_dirオプション
├── pycrop
│   └── --referenceオプション
│       ├── 00000.avi
│       ├── 00001.avi
│       ├── 00002.avi
│       ├── ...
│       └── xxxxx.avi
└── pywork
    └── --referenceオプション
        ├── faces.pckl
        ├── scene.pckl
        └── tracks.pckl
```

次は`run_syncnet.py`で動画の音声が映っている人物から発せられたのか、その確からしさのもととなる誤差をNNで計算します。
`run_pipeline.py`と同一の`--date_dir`オプションと`--reference`オプションを指定します。
スクリプトが正常終了すると、誤差?情報が`pywork/${--reference}/activesd.pckl`に保存されます。

`run_syncnet.py`の結果`activesd.pckl`から、各顔動画のスコア(Confidence)を計算します。
`syncnet_python`のソースコードから、それらしい処理を見つけたのでそのまま使います(数学難しい並感)。

{% highlight python linenos %}
def calculate_confidence(activesd_path):
    with open(activesd_path, 'rb') as fp:
        dists = pickle.load(fp)
    confidences = []
    for dist in dists:
        mean_dists = np.mean(dist, 0)
        minval = np.min(mean_dists)
        conf = np.median(mean_dists) - minval
        confidences.append(conf)
    return confidences
{% endhighlight %}

`calculate_confidence`は`run_syncnet.py`の結果ファイル`activesd.pckl`から、各動画のスコアを計算します。スコアが高いほどsyncnetによってより高品質なデータだと推定された動画であることを示します。

### 3 Confidenceのヒストグラム

Google Colaboratoryを使って3日ほど収集したデータに対するConfidenceのヒストグラムを示します。

![](/assets/img/youtuberer/day-5-fig-1-1.png)

経験則的にConfidenceが5を越えるとデータセットに採用できる品質になるのですが、その数はまだまだ少ないです。

### 4 次回予告

まとまった量集まったら顔画像認証を使って日本語版VoxCeleb作成の実験をします。

### 追記

ソースコードを[こちら](https://github.com/leichtrhino/youtuberer)に公開しました。
