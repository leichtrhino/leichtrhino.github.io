---
layout: post
title:  "Chimeraの系譜 その2"
date:   2020-04-27 01:00:00 +0900
mathjax: true
---

[前回]({% post_url 2020-04-18-chimeranet-family-day-1 %})

phasebook, combookの個人用まとめ(と草生やし)です。

### 前回からの進捗まとめ

1. phasebook, combookを調査
2. MISIレイヤーとphasebook, combookの実装 (TBA)
3. DSD100データセットの使用 (TBA)

### 1. phasebook, combookの調査

* J. L. Roux, G. Wichern, S. Watanabe, A. Sarroff and J. R. Hershey, "The Phasebook: Building Complex Masks via Discrete Representations for Source Separation," ICASSP 2019 - 2019 IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), Brighton, United Kingdom, 2019, pp. 66-70.
* arXiv:1810.01395 (上とは別の論文です)

前回のconvex softmaxでは、入力\\(X\\)に対する各マスクの`(時間, 周波数)`での要素値\\(m^{out}_{t,f}\\)を次の方法で求めていました。

\\[m^{out}\_{t,f}=\\sum\_\{i=0\}^2 i\\:p(m\_{t,f}=i\|X)\\]

ここで、\\(m\_{t,f}\\)は離散化されたマスクの値を表す確率変数で、\\(\\{0,1,2\\}\\)
のいずれかの値をとります。

**phasebook**は離散的な値と確率分布を使って位相の予測を試みています。
\\(m\\)を\\(\\theta\\)に読みかえ、確率変数\\(\\theta\_{t,f}\\)がとりうる離散値を\\(\\theta^{\(i\)}=2\\pi i/8 \\:\(i\\in\\{1,\\ldots,8\\}\)\\)とすると、

\\[\\theta^{out}\_{t,f}=\\sum\_\{i=1\}^8 \\theta^{\(i\)}p(\\theta\_{t,f}=\\theta^{\(i\)}\|X)\\]

によって求めることができます。

\\(\\theta^{out}\_{t,f}\\)を期待値によって求める他にも、argmax的に求める、確率分布に従って生成する方法もあります(が、めんどくさいので補間のみ実装します。

\\(m^{out}\_{t,f}\\)と\\(\\theta^{out}\_{t,f}\\)を求めると、\\(c^{out}\_{t,f}=m^{out}\_{t,f}e^{j\\theta^{out}\_{t,f}}\\)として複素数領域のマスクが求められます。

以前の誤差関数を拡張して、complex mask approximationの目的関数は、\\(c^{ref}\_{t,f}=s\_{t,f}/x\_{t,f}\\)など適当なマスクとして、
\\[\\mathcal{L}\_{CMA,L^1}\(\\phi\)=\\sum\_{t,f}\|c^{out}\_{t,f}-c^{ref}\_{t,f}\|\\]
、また、complex spectrum approximationの目的関数は、
\\[\\mathcal{L}\_{CMA,L^1}\(\\phi\)=\\sum\_{t,f}\|c^{out}\_{t,f}x\_{t,f}-s\_{t,f}\|\\]
とします。

一方**combook**は、magbook, phasebookの考え方を複素数に拡張して、複素数のマスクを直接求めます。
\\(C\\)個の複素数\\(\\{c^{\(1\)},\\ldots,c^{\(C\)}\\}\\)を使って複素数マスクの\\(\(t,f\)\\)成分を

\\[c^{out}\_{t,f}=\\sum\_\{i=1\}^8 c^{\(i\)}p(c\_{t,f}=c^{\(i\)}\|X)\\]

として求めます。なお、実装したcombookは複素数集合\\(\\{c^{\(1\)},\\ldots,c^{\(C\)}\\}\\)を他のパラメータと共に学習します。

### 2. MISIレイヤーとphasebook, combookの実装

TBA

### 3. DSD100データセットの使用

TBA

### 次回予告

評価方法を調査し、実装します。さらに、その方法を用いて学習したモデルを評価します。
