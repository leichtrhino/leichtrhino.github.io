---
layout: post
title:  "あつまれ YouTuberのデータ その2"
date:   2020-05-01 22:00:00 +0900
---

[前回]({% post_url 2020-04-29-youtuberer-day-1 %})

### 今回の目標

1. 前回のプログラムの整理(当該ページも更新します)
2. 顔検出ライブラリによる動画の選別

### 1. 前回のプログラムの整理(当該ページも更新します)

#### 簡単な集計

次のスクリプトは収集したjsonをもとに、時間、検索したカテゴリID、トレンド内のランク、動画ID、チャンネルID、カテゴリID、タイトル、チャンネル名のカンマ区切りファイルを作成します。

次の関数はDriveに保存されたメタデータのgeneratorです。

{% highlight python linenos %}
import os
import json

def video_items(timestr_from=None, timestr_to=None):
  if timestr_from is None:
    timestr_from = ''
  if timestr_to is None:
    timestr_to = 'A'
  base_dir = '/content/drive/My Drive/YouTubeTrend/'
  items_per_page = 50
  for d, f in [
               (d, f)
               for d in sorted(os.listdir(base_dir))
               if os.path.isdir(os.path.join(base_dir, d)) and
               timestr_from < d < timestr_to
               for f in os.listdir(os.path.join(base_dir, d))
               ]:
    jsonpath = os.path.join(base_dir, d, f)
    if len(os.path.splitext(f)[0].split('-')) != 3:
      continue
    scatid, ipage = tuple(map(int, os.path.splitext(f)[0].split('-')[1:]))
    with open(jsonpath, 'r') as fp:
      jsonobj = json.load(fp)
      for rank_in_page, item in enumerate(jsonobj['items'], 1):
        yield d, scatid, (ipage-1)*items_per_page+rank_in_page, item
{% endhighlight %}

### 2. 顔検出ライブラリによる動画の選別

#### なぜサムネイルを顔検出にかけるのか

目的(LipSyncで日本語版VoxCelebを作る)の観点から、サムネイルに顔が映っている動画(あるいはそのチャンネルの動画)を残せば効率的であることが予想されます。

そこで、このスクリプトによって動画のサムネイルを取得し、画像に顔画像検出処理をかけます。

Lucas Persona氏の[Notebook](https://colab.research.google.com/drive/1lJWquGmKoMm68qNuwjSnfMjjIi-UTzI1)に従って処理します。
まずはライブラリのインストールから

{% highlight python linenos %}
%tensorflow_version 1.x
import tensorflow
!pip install -q -U imutils git+https://github.com/the-house-of-black-and-white/hall-of-faces.git
{% endhighlight %}

`hof`と呼ばれるライブラリに実装された``SSDMobileNetV1FaceDetector``を用いて顔の位置を検出します。
`SSDMobileNetV1FaceDetector`の他に、`RfcnResnet101FaceDetector`, `TinyYOLOFaceDetector`などがあります。

{% highlight python linenos %}
from imutils import  url_to_image
from hof.face_detectors import SSDMobileNetV1FaceDetector

MIN_CONFIDENCE = 0.8
face_detector = SSDMobileNetV1FaceDetector(min_confidence=MIN_CONFIDENCE)
def obtain_faces_in_thumbnail(item):
  thumbnails = item['snippet']['thumbnails']
  resolution = 'maxres' if 'maxres' in thumbnails else \
    'standard' if 'standard' in thumbnails else 'high'
  try:
    thumbnail = url_to_image(item['snippet']['thumbnails'][resolution]['url'])
  except:
    return []
  face_boxes = face_detector.detect(thumbnail)
  return face_boxes
{% endhighlight %}

上の関数は動画のメタデータ(Videos)からサムネイルを取得し顔検出処理にかけて
その結果を返すものです。
この関数を使って、保存されたメタデータから、サムネイルに顔が映っている動画のみを
抜き出すことができます。

{% highlight python linenos %}
base_dir = '/content/drive/My Drive/YouTubeTrend/'
ofp = open(os.path.join(base_dir, 'contain-face.csv'), 'w')
print('time,scatid,rank,id,cid,catid,title,ctitle,nfaces', file=ofp)

for d, scatid, rank, item in video_items():
  faces = obtain_faces_in_thumbnail(item)
  print(','.join(map(str, (
    d, scatid, rank,
    item['id'], item['snippet']['channelId'],
    item['snippet']['categoryId'],
    '"{}"'.format(item['snippet']['title'].replace('"', '""')),
    '"{}"'.format(item['snippet']['channelTitle'].replace('"', '""')),
    len(faces)
  ))), file=ofp)

ofp.close()
{% endhighlight %}

### 次回予告

現在実行中なので、実行が完了次第、集計をします。
その結果をもとに、LipSync処理に移るか、さらにメタデータを集計するか判断します。

[^1]: http://www.robots.ox.ac.uk/~vgg/data/voxceleb/
