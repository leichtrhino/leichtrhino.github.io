---
layout: post
title:  "ChimeraNet+x-vectorでデータを集めたい その3"
date:   2020-04-17 06:00:00 +0900
---

[前回]({% post_url 2020-04-13-chimeranet-xvector-day-2 %})

続きです。

日曜夕方から計算を開始し、木曜朝に一通り終わりました。
しかし一部エラーで計算できていなかったので、その部分のみ再計算し、結局夕方に完了しました。

いきなりですが実験結果を示します。

### 実験1 マスクが与える話者照合における影響

以下の表はspk=Serval,n-samples=1000で学習した各PLDAの等価エラー率(%)を表します(各列の最小値は**太字**で表示)。
行方向に学習セットを、列方向にテストセットを表します。
表の記号については前の記事をご覧ください。
なお、テストセットは`ABC`と`D`共通で、それぞれ100個の音声区間を使用しています。
単一のパラメータに対して20回繰り返しました。

|            | raw,ABC  | raw,D     | mask,ABC  | mask,D    | embd,ABC  | embd,D    |
| :--------- | -------: | -----:    | --------: | ------:   | --------: | ------:   |
| raw,ABC    | **7.6**  | **19.14** | **9.52**  | 17.72     | 19.36     | 27.82     |
| mask,ABCD  | 12.4     | 21.91     | 11.21     | **17.03** | 20.62     | 28.98     |
| embd,ABCD  | 14.7     | 24.37     | 14.00     | 19.84     | **15.68** | **22.71** |

テストセットのraw(マスクなし)について、rawで学習したPLDAモデルの性能が最も性能が良いことがわかります。
テストセットmask(softmaxで求めたマスク)に対して、品質ABCに対してはrawセットで学習したPLDAの性能が最も高いことは意外でした。
しかし、品質Dに対しては、わずか0.7ポイント差ですが、maskセットで学習したPLDAモデルの方が良いことがわかります。

### 実験2 サンプル数の変化に伴う話者照合の性能の変化

サンプル数を変化させたときのEERの変化を調べます。
サンプル数以外は、実験1の設定と同じです。

実験結果は以下です。横軸はサンプル数を、縦軸はEER(単位は%)を表します。
さらに、縦軸方向の点の位置は20試行の平均値を、各点から上下に伸びた線は標準偏差を表します。
なお、点および線の色は学習セットとテストセットの組合せを表し、凡例では`train -> test`のように表しています
(例: `raw,ABC -> mask,D`は学習セットとして`raw,ABC`を、テストセットとして`mask,D`を用いたことを表します)。

![サンプル数変化時のEERの変化](/assets/img/chimeranet-xvector/eer-sample-size.png)

見た感じサチってない気がします。この特徴は`raw,ABC -> raw,ABC`(ground truth的な一番いいやつ)でも見られます。
より精度を高めるにはもっとデータが必要ということでしょうか。なかなか手強い困りものですね。

全体の傾向として、テストセット`mask,D`に対するEERが最も高く、頭ひとつ抜けています(ちゃんと検定していませんが)ので、
信号分離の性能ももっとよくしたいですね。

### 次回予告

シリーズは一旦お休みして、より高い性能の信号分離モデルの実装に取り掛かります。
