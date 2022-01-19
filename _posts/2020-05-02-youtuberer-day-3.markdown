---
layout: post
title:  "あつまれ YouTuberのデータ その3"
date:   2020-05-02 06:00:00 +0900
mathjax: true
---

[前回]({% post_url 2020-05-01-youtuberer-day-2 %})

今日は収集したデータを集計し、次の行動を決めます。 ちなみにこれは`rmarkdown`を用いて作成しています。
ソースコードは[こちら](https://github.com/leichtrhino/leichtrhino.github.io/tree/master/assets/Rmd/2020-05-02-youtuberer-day-3.Rmd)

### 使用ライブラリ、データの読み込み

次のライブラリを読み込みます。 最後の部分は`ggplot`の設定です。

``` r
library(readr)
library(dplyr)
library(tidyr)
library(forcats)
library(ggplot2)
library(viridis)
library(knitr)
theme_set(theme_light(base_size = 12, base_family = 'IPAexGothic'))
```

以下のデータを読み込み、テーブルの結合を行います。

``` r
categories <- read_csv('categories.csv') %>%
    rename(cattitle = title)
```

    ## Parsed with column specification:
    ## cols(
    ##   id = col_double(),
    ##   title = col_character()
    ## )

``` r
data <- read_csv('contain-face.csv') %>%
    left_join(categories, by = c('catid' = 'id')) %>%
    mutate(cattitle = fct_reorder(cattitle, catid))
```

    ## Parsed with column specification:
    ## cols(
    ##   time = col_character(),
    ##   scatid = col_double(),
    ##   rank = col_double(),
    ##   id = col_character(),
    ##   cid = col_character(),
    ##   catid = col_double(),
    ##   title = col_character(),
    ##   ctitle = col_character(),
    ##   nfaces = col_double()
    ## )

### 基本的な集計

トレンドに載っていた動画は3060本、 チャンネルの総数は1873です。

次の図はカテゴリごとの動画の本数を表します。

``` r
data %>%
    select(-c(time, scatid, rank)) %>% # 同一の動画の除去
    distinct() %>%
    ggplot(aes(x=cattitle)) +
    geom_bar() +
    theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))
```

![](/assets/img/youtuberer/day-3-fig-1-1.png)<!-- -->

### 特定カテゴリの人気の動画

各カテゴリの人気の動画を見ていきましょう。 次のコードによってカテゴリ名`Entertainment`で上位5個に
入ったことがある動画およびそのチャンネル名、最も上位のランクを表示します。
なお、“上位”について、`Data API`で得られた結果に基づき求めているので、
必ずしも正確なデータではありません。ご留意ください。

``` r
data %>%
    filter(cattitle == 'Entertainment') %>%
    group_by(id) %>%
    summarise(minrank = min(rank)) %>%
    filter(minrank <= filter_rank) %>% # filter_rank: 本文中にこっそり定義されています
    inner_join(
        data %>% select(c(id, title, ctitle)) %>% distinct(),
        by = c('id' = 'id')
    ) %>%
    arrange(minrank) %>%
    select(c(minrank, title, ctitle)) %>%
    kable
```

```
諸事情により結果は割愛させていただきます
```

近年話題のプレイ動画などが含まれる`Gaming`についても同様に。

```
諸事情により結果は割愛させていただきます
```

`Entertainment`と`Gaming`を比較するのも面白いでしょう
（トレンドの動画の数と“特に”トレンドの動画の数の割合が一致しない理由を考えるなど）。

### カテゴリの関係（しません）

各カテゴリの関係をチャンネルの観点から見つけます。
ここでは条件付き確率\\(p(c\_{i}|c\_{j})=p(c\_{i},c\_{j})/p(c\_{i})\\)
を求めます。ここで\\(c\_{i}\\)は\\(i\\)番目カテゴリを、\\(p(c\_{i})\\)はあるチャンネルがカテゴリ
\\(c\_{i}\\)の動画をアップロードしている確率を、さらに、\\(p(c\_{i},c\_{j})\\)は
あるチャンネルがカテゴリ\\(c\_{i}\\)の動画とカテゴリ\\(c\_{j}\\)の動画をアップロードしている確率を
表します。

と思いましたが、1873個のチャンネルのうち、異なるカテゴリの動画を アップロードしているものは43個
しかなかったので、もう少し集まってから試してみたいと思います。

### サムネイルに顔が載っている動画について

ここからが本題です。最終目標は日本語版VoxCeleb的なデータセットの作成で、
効率的な収集のために、顔が映っている動画を優先的に処理することが不可欠です。

これを実現するひとつの方法として、サムネイルに顔が載っている動画、および
そのような動画をアップロードしているチャンネルの動画のリストアップを考えました。

ここでは、収集したメタデータをもとに、サムネイルに顔が映っている動画がどれくらいの 割合であるのか調べます。

#### 基本

サムネイルに顔が映っている動画は 3060本中 1119本で、 その割合は36.5686275%。

また、顔が写っている動画がアップロードされているチャンネルは、全1873チャンネル中、 863チャンネルで、 その割合は46.0758142%。

#### カテゴリごとに

次のコマンドでカテゴリごとの顔含有数の図を描画します。

``` r
data %>%
    select(c(id, cattitle, nfaces)) %>%
    distinct() %>%
    mutate(has_face = nfaces > 0) %>%
    ggplot(aes(x = cattitle, fill = has_face)) +
    geom_bar() +
    scale_fill_viridis(discrete = TRUE, begin = 0.3, end = 0.7) +
    theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))
```

![](/assets/img/youtuberer/day-3-fig-2-1.png)<!-- -->

`Pets & Animals`の含有率が極端に低いのですが、人間載せても意味ないから当たり前か。

### まとめと次回予告

1000本を超える顔画像サムネイル動画と、およそ900個のチャンネルが見つかったので、
引き続き、全体のメタデータの収集を進めつつ、分析対象の動画の選定に移ります。
