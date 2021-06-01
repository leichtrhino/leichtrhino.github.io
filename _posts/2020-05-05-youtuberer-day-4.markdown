---
layout: post
title:  "あつまれ YouTuberのデータ その4"
date:   2020-05-05 014:00:00 +0900
published: false
---

[前回]({% post_url 2020-05-02-youtuberer-day-3 %})

今日は与えられたチャンネルから、1) アップロードされた動画のサムネイルの顔画像検出、 2) タイトルから言語の予測を行います。

こちらも`rmarkdown`を用いて作成しています。ソースコードは[こちら](https://github.com/leichtrhino/leichtrhino.github.io/tree/master/assets/Rmd/2020-05-05-youtuberer-day-4.Rmd)

### 目次

1.  データの準備
    1.  動画リストの取得
    2.  サムネイルの顔画像検出
    3.  タイトルの言語推定
2.  分析
    1.  サムネイルに顔が映っている動画の割合
    2.  チャンネル・動画の言語の分布
    3.  顔サムネイル、日本語の割合
3.  まとめと次回予告

### 1\. データの準備

以下の手順によりチャンネルがアップロードした動画の情報を取得します。

1.  動画リストの取得
2.  サムネイルの顔画像検出
3.  タイトルの言語推定

プログラムは今後整理して需要がありそうなら公開します。

#### 1\. 動画リストの取得

与えられたチャンネルから、そこにアップロードされた動画のリストを取得します。 当初は`YouTube Data
API`を使う予定でしたが、目的にあった操作は`Search`しかなく、
`Search`は重い処理(quotaの都合上、執筆時点で一日100回が限度)なので
`Selenium`を使います(本来とは違った目的ですが)。

この処理により、チャンネルのリストから、いちチャンネルにつき数十本の動画のIDとサムネイルのURL、 タイトルを取得します。

#### 2\. サムネイルの顔画像検出

当初の目的と変わらず、効率的に進めるため動画のサムネイルに顔が映っているか判定します。
前回は[HoF](https://github.com/the-house-of-black-and-white/hall-of-faces)
を使ったのですが、今回は将来使うであろう[SyncNet](https://github.com/joonson/syncnet_python)と同じバックエンド(PyTorch)を使った[facenet-pytorch](https://github.com/timesler/facenet-pytorch)を使います。

このfacenet-pytorchは顔認証的なこともできそうなので、SyncNetの後の処理でも使います。

この処理により、取得された動画サムネイルのURLに顔が映っているかを判定します。

#### 3\. タイトルの言語推定

分析途中、日本語ではない動画チャンネル(世界的なモータースポーツイベント、海外を活動拠点とするアーティストなど)が含まれており、
日本語版VoxCelebの作成のためにフィルターする必要があります。

調べると、多言語テキストの分析ライブラリ[polyglot](http://polyglot-nlp.com)が見つかったので、これを用います。

この処理により、動画のタイトルからその動画の言語を推定します。

### 2\. 分析

以下、取得したデータを分析し、

1.  サムネイルに顔が映っている動画の割合
2.  チャンネル・動画の言語の分布
3.  顔サムネイル、日本語の割合

の各項目の調査を通じて、分析対象とするチャンネルの選定を行います。 なお、501個のチャンネル、 32380本の動画を対象とします。

#### 1\. サムネイルに顔が映っている動画の割合

サムネイルに顔が映っている動画は全32380本中 18134本で その割合は56.004%です。

また、サムネイルに顔が映った動画を1本以上アップロードしているチャンネルは 全501チャンネル中 498本で その割合は99.401%です。

あれ?結構多い…誤認識かな?

チャンネルの顔サムネイル含有率のヒストグラムは以下です。

``` r
data %>%
    mutate(has_face = n_faces > 0) %>%
    group_by(cid) %>%
    summarise(face_frac = sum(has_face) / n()) %>%
    ungroup() %>%
    ggplot(aes(x = face_frac)) +
    geom_histogram()
```

![](/assets/img/youtuberer/day-4-fig-1-1.png)<!-- -->

あれ、顔を見せない/見せるで双峰的になると思ってたけど、その特性弱そう。

gmmで検証してみる。[ここ](https://stackoverflow.com/questions/25313578/any-suggestions-for-how-i-can-plot-mixem-type-data-using-ggplot2)を参考にしました。

``` r
library(mixtools)
data_hist <- data %>%
    mutate(has_face = n_faces > 0) %>%
    group_by(cid) %>%
    summarise(face_frac = sum(has_face) / n()) %>%
    ungroup()
mixture <- normalmixEM(data_hist %>% pull(face_frac))
```

    ## number of iterations= 67

``` r
sdnorm <- function(x, mean=0, sd=1, lambda=1) {
    lambda * dnorm(x, mean=mean, sd=sd)
}
ggplot(data_hist) +
    geom_histogram(
        aes(x = face_frac, y = ..density..),
        fill = 'gray80', colour = 'gray50') +
    stat_function(fun = sdnorm,
                  args = list(mean = mixture$mu[1],
                            sd = mixture$sigma[1],
                            lambda = mixture$lambda[1]),
                  colour = 'orange', geom = 'line', size = 2) +
    stat_function(fun = sdnorm,
                  args = list(mean = mixture$mu[2],
                            sd = mixture$sigma[2],
                            lambda = mixture$lambda[2]),
                  colour = 'royalblue', geom = 'line', size = 2)
```

![](/assets/img/youtuberer/day-4-fig-2-1.png)<!-- -->

顔出す人はほぼ毎回出すけど、それ以外はぼちぼち?全然わからん

#### 2\. チャンネル・動画の言語の分布

動画のタイトルで検出された言語は c(“ja”, “en”, “ln”, “tl”, “da”, “sk”, “wo”, “it”, “co”,
“ia”, “sn”, “ny”, “eu”, “pt”, “om”, “war”, “zh\_Hant”, “fr”, “vi”, “id”,
“jw”, “ms”, “sco”, “la”, “ca”, “kl”, “na”, “cs”, “mi”, “ru”, “es”, “so”,
“un”, “ceb”, “rw”, “st”, “zzp”, “fy”, “sq”, “br”, “ko”, “nn”, “ht”,
“fj”, “tlh”, “sa”, “rn”, “za”, “gl”, “vo”, “ro”, “zh”, “kha”, “af”,
“mfe”, “zu”, “hi”, “hu”, “su”, “bi”, “th”, “ig”, “ha”, “no”, “ne”,
“bh”, “mr”, “cy”, “mg”) です。

チャンネル内で頻出している言語(いわば、チャンネルの言語)の分布を次の図に示します。

``` r
data %>%
    group_by(cid) %>%
    count(language) %>%
    top_n(1) %>%
    ungroup() %>%
    group_by(language) %>%
    summarise(count = n()) %>%
    ggplot(aes(x = reorder(language, -count), y = count)) +
    geom_bar(stat = 'identity') +
    theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))
```

    ## Selecting by n

![](/assets/img/youtuberer/day-4-fig-3-1.png)<!-- -->

やはりほとんどのチャンネルが日本語の動画をメインで配信しているようです。

#### 3\. 顔サムネイル、日本語の割合

顔サムネイルが80%以上含まれる日本語のチャンネルは 全501チャンネル中 112チャンネルで、その割合は22.355%です。

### 3\. まとめと次回予告

与えられたチャンネルの一覧から、分析対象となるチャンネルをフィルタリングする方法を 検討し、簡単に実験しました。
次回は、SyncNetの使い方をメインに書きます。
