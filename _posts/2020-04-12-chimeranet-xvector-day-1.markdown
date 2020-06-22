---
layout: post
title:  "ChimeraNet+x-vectorでデータを集めたい その1"
date:   2020-04-12 23:00:00 +0900
---

[次回]({% post_url 2020-04-13-chimeranet-xvector-day-2 %})

本シリーズは[記事](https://leichtrhino.hatenablog.com/entry/2020/06/22/233503)の続きで、タイトルの通りChimeraNetでBGMを減衰させた音声のx-vectorを計算して分析します。
今日は評価用データセットの説明を行います。

### データセット
以前作成したデータセットに品質`D`(BGMがかかっている)を追加し、
これに対する音声区間を手動で収集しました。

収集は始業前、終業後の時間を使ってちまちま行いました。
自由ではなくなったので、時間を有効に使う意識を持って取り組めますね。

### ヒストグラム
データセットをもとに以下のヒストグラムを描きました。
やはり主人公二人に偏ってますね。
実験でもこの二人を対象にしたいと思います。

![ヒストグラム](/assets/img/chimeranet-xvector/speaker-histogram.png)
