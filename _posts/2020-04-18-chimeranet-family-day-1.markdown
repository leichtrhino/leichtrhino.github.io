---
layout: post
title:  "Chimeraの系譜 その1"
date:   2020-04-18 23:50:00 +0900
---

### シリーズの目的

* 信号分離を行う深層学習モデルのひとつ`Chimera`とその派生を調査し実装する

### 調査その1

ここからは完全に個人のメモ帳です。私自身プロでないので、きっと間違っています。あてにしちゃあだめ。

0. 共通の問題設定
  - 合成信号のスペクトログラムから合成前の信号を推定する
1. Deep clusteringに基づく信号分離
  - arXiv:1508.04306
  - 合成信号のスペクトログラムの一点(t,f)のデータを20次元程度に埋め込み
  - 埋め込み表現と訓練データの次元は一致しないので新たな目的関数を定義
  - 埋め込み表現をクラスタリングすると、同一ラベルの点は同一信号元の点になっている
2. Chimera
  - arXiv:1611.06265
  - deep clusteringに基づく推定とマスクの推定の合わせ技(マルチタスク学習)
  - マスクの推定は信号源間の分離を良くし、deep clusteringは同一信号元内の分散を小さくする働き
  - おそらく、後続のモデルの基本
3. Chimera++
  - Wang, Zhong-Qiu, Jonathan Le Roux, and John R. Hershey. "Alternative objective functions for deep clustering." 2018 IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP). IEEE, 2018.
  - 従来のdeep clusteringに代わる誤差関数を定義
  - whitened k-meansがよく使われる印象
  - マスクの推定を埋め込み表現からではなく、その前のBiLSTMから推定させる
  - さらに、マスク推定の層に対する誤差関数も新たに追加(truncated phase-sensitive spectrum approximation)
4. multiple input spectrogram inverseによる位相の復元
  - arXiv:1804.10204
  - MISI法による位相の復元をChimera++のマスク推定結果の後に追加
  - MISI法によって得られた各チャネルの信号と、正解の信号の誤差も定義されている
  - MISI法がもともとマスクにより分離された信号を対象にしていたため、親和性が高い
  - MISI法に加えて、マスク推定層に新たな活性化関数の導入を提案
    * マスク中の一点に対し、そこでの値が(0, 1, 2)それぞれである確率をsoftmaxで計算し
    * 期待値をその点の値とする
    * のちのphasebook、magbookの考えに近い

### 実装その1

#### 方針

まず、MISI法による位相復元つきChimera++の実装を完了させる。
その後、phasebook、magbookを実装したい

#### 今日の成果

* プライベートリポジトリを立ち上げ、基本的な誤差関数とそれに対するテストを実装

