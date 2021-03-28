---
layout: post
title:  "あつまれ YouTuberのデータ その6"
date:   2021-03-28 20:00:00 +0900
---

[前回]({% post_url 2020-06-03-youtuberer-day-5 %})

前回は`syncnet-python`を使って、YouTube動画内で人が喋っている区間を切り出しました。
しかし、SyncNetのConfidence単体では効果音やBGMが入っているかどうかを判別することはできませんでした（結果を聴いて確認しました）。
このデータセットの活用先として、楽器の音と人の声を分離することがあるのですが、効果音やBGMが入っていると望ましくない結果が得られると予想されます。
そのため、第6回は人の声に対する雑音がどの程度含まれているかを推定する[`snreval`](https://labrosa.ee.columbia.edu/projects/snreval/)を使って、動画の区間が人の声のみかどうかを推定します。

### 目次

1. `snreval`の使用法
2. 各スコアを一括で推定する
3. 一般線形モデルでBGMがないか推定する
4. 次回予告

### 1 `snreval`の使用法

`snreval`は、音声の品質を推定するためのアルゴリズムを実装したMATLABの関数群です。
リンク先にあるように、`NIST STNR`, `WADA SNR`などの評価方法を実装しています。

ダウンロードするには、リンク先のInstallationセクションに記載されたリンクから、環境に合わせてダウンロードします
（MATLAB向けにコンパイルされたパッケージか、ソースコードから選択できます）。

ここでは、GNU Octave 6.1.0を対象とします。ソースコードをダウンロード・展開した後、
`audioread.m`を削除します（Octave 6.1.0で`audioread`は標準で使用できるため）。
Octaveを起動し、`pkg load signal`で必要なパッケージを読み込みます。
カレントディレクトリを展開先に合わせたら、次のコマンドを実行し、`snreval`を呼び出します。

```
[SNRstnr,SNRwada,SNRvad,SAR,pesqmos,targdelay] = ...
snreval('waveform.wav', '-samplerate', 16000, '-guessvad', 1, '-disp', 0);
```

`waveform.wav`は分析対象のファイル名を表します。
`-samplerate`直後の数値を指定すると、ファイルを指定したサンプリング周波数にしてから分析します。
NIST STNR, WADA SNR, SNR-VADそれぞれの結果が`SNRstnr`, `SNRwada`, `SNRvad`に格納されます。
それぞれ値が大きいほど、品質が高いと言えます。

### 2 各スコアを一括で推定する

前回抽出した動画ファイルに対して一括で評価する関数を`batch_process.m`に作成しました。

{% highlight matlab linenos %}
function batch_process (root_directory, output_file)
  video_group_list = glob(cstrcat(root_directory, '/*'));
  proc_video_group_num = rows(video_group_list);
  processed_video_num = 0;
  
  sc = cell(100, 4);
  sc{1, 1} = 'scene';
  sc{1, 2} = 'nist-stnr';
  sc{1, 3} = 'wada-snr';
  sc{1, 4} = 'snr-vad';
  
  for j = 1:proc_video_group_num
    file_list = glob(cstrcat(video_group_list{j}, '/*.avi'));
    if rows(file_list) == 0
      continue;
    endif
    prev_processed_video_num = processed_video_num;
    
    for i = 1:rows(file_list)
      f = file_list{i};
      system(cstrcat(
        '/usr/local/bin/ffmpeg -loglevel quiet -y -i "', f, '" -vn -acodec copy tmp.wav'
      ));
      disp(f);
      [SNRstnr,SNRwada,SNRvad,SAR,pesqmos,targdelay] = ...
        snreval('tmp.wav', '-samplerate', 16000, '-guessvad', 1, '-disp', 0);
      sc{prev_processed_video_num+i+1, 1} = f;
      sc{prev_processed_video_num+i+1, 2} = SNRstnr;
      sc{prev_processed_video_num+i+1, 3} = SNRwada;
      sc{prev_processed_video_num+i+1, 4} = SNRvad;
    endfor
  endfor
  cell2csv(output_file, sc);

endfunction
{% endhighlight %}

`batch_process('/path-to-syncnet-out/pycrop', 'score-snr.csv')`を呼び出すと、
`/path-to-syncnet-out/pycrop`以下のファイルに対して、各種スコアを計算して`score-snr.csv`に保存します。

`/path-to-syncnet-out/pycrop`の構造は以下の通りです。

```
--path-to-syncnet-out
└── pycrop
    └── 動画ID
        ├── 00000.avi
        ├── 00001.avi
        ├── 00002.avi
        ├── ...
        └── xxxxx.avi
```

`score-snr.csv`は、次の列をもったcsvファイルです。

- `scene`: 動画ID/xxxxx.avi
- `nist-stnr`: NIST STNR
- `wada-snr`: WADA SNR
- `snr-vad`: SNR VAD

### 3 一般線形モデルでBGMがないか推定する

#### データセットの準備

前回`syncnet`で算出したスコアと`snreval`で算出したスコアから、音声に効果音・BGMが入っていないか推定するための一般線形モデルを構築します。

おさらいですが、`syncnet`で算出したスコアは以下の項目をもちます。
このファイルを`score-syncnet.csv`とします。

- `scene`: 動画ID/xxxxx.avi
- `score`: SyncNetのスコア
- `length`: 動画の長さ(s)

また、推定のために、次のデータを作成しておきます。
このファイルを`clean.csv`とします。

- `scene`: 動画ID/xxxxx.avi
- `t/f`: 音声に効果音・BGMが含まれていない: `TRUE`, 含まれている: `FALSE`

#### 一般線形モデルの作成・評価

まず、全ての項目を集めた`score.csv`と、モデル作成に必要な項目を集めた`score-tf.csv`を作成します。

{% highlight R %}
library(readr)
library(dplyr)
score_syncnet <- read_csv('score-syncnet.csv')
score_snreval <- read_csv('score-snreval.csv')
score <- score_syncnet %>%
  inner_join(score_snreval) %>%
  mutate(video = gsub('/.*', '', scene))
score_by_video <- score %>%
  group_by(video) %>%
  summarise(
    `mean-score` = sum(score*length) / sum(length),
    `mean-nist-stnr` = sum(`nist-stnr`*length) / sum(length),
    `mean-wada-snr` = sum(`wada-snr`*length) / sum(length),
    `mean-snr-vad` = sum(`snr-vad`*length) / sum(length)) %>%
  ungroup()
score <- score %>% inner_join(score_by_video)
write_csv(score, 'score.csv')

tf <- read_csv('clean.csv') %>% filter(!is.na(`t/f`))
tf %>% inner_join(score) %>% write_csv('score-tf.csv')
{% endhighlight %}

#### 一般線形モデルを用いて推定

次に、`score.csv`の内容をもとに一般線形モデルを作成し、混合行列を求めます。
その後、全サンプルを用いてモデルを作成し、推定結果を`prediction.csv`に書き出します。

{% highlight R %}
library(readr)
library(dplyr)
set.seed(456)

score <- read_csv('score.csv')
tf_all <- read_csv('score-tf.csv') %>%
  mutate(
    `snr-vad` = pmax(-10000, `snr-vad`),
    `mean-snr-vad` = pmax(-10000, `mean-snr-vad`))
tf_train <- sample_frac(tf_all, 0.8)
tf_test <- anti_join(tf_all, tf_train, by = c('scene'))

model <- glm(
  data = tf_train,
  `t/f` ~ score + length + `nist-stnr` + `wada-snr` + `snr-vad` +
          `mean-score` + `mean-nist-stnr` + `mean-wada-snr` + `mean-snr-vad`,
  family=binomial())
table(
  tf_test$`t/f`,
  predict(model, tf_test, 'response') >= 0.5,
  dnn = c('test', 'prediction'))

model_all <- glm(
  data = tf_all,
  `t/f` ~ score + length + `nist-stnr` + `wada-snr` + `snr-vad` +
          `mean-score` + `mean-nist-stnr` + `mean-wada-snr` + `mean-snr-vad`,
  family=binomial())
score %>% mutate(
  prediction = predict(
    model_all,
    mutate(
      .,
      `snr-vad` = pmax(-10000, `snr-vad`),
      `mean-snr-vad` = pmax(-10000, `mean-snr-vad`)
    ),
    'response') >= 0.5
) %>% select(c(scene, prediction)) %>% write_csv('prediction.csv')
{% endhighlight %}

### 4 次回予告

（ある程度）きれいな音声データが抽出できたので、次は顔画像と合わせて話者推定データセットの作成に取り組みます。
