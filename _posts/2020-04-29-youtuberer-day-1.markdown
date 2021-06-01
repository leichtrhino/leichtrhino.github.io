---
layout: post
title:  "あつまれ YouTuberのデータ その1"
date:   2020-04-29 20:00:00 +0900
published: false
---

### 長期目標

* 日本語版VoxCelebを作りたい
  - VoxCelebの作り方は公式サイト[^1]の論文に載っている
  - SyncNetという技術を用いて、顔と発話を紐付け
  - VGGFaceで提供されている顔画像照合によって、話者照合の質を保証
* そのためには、YouTubeに動画を投稿するYouTuberの情報の収集は不可欠です。
* 今日はYouTubeに投稿されたトレンドの動画のメタデータを収集します。

例によってこれはメモ書きなので、後日書き直します。

### 目次

1. Google apps scriptで定期的に収集
2. Google Colabで集計

### 1. Google apps scriptで定期的に収集

以下のスクリプトをGoogle apps scriptとして実装し、
一時間ごとに実行するようトリガをかけました。

apps script側から、Google Cloud Platformへの関連づけが必要です。
さらに、Google Cloud Platform側でDriveとYouTube data api v3を有効にする必要があります。

まだ下書き程度ですので、完成したら解説をつけます。

{% highlight javascript linenos %}
function myFunction() {
  var folder = DriveApp.getFoldersByName('YouTubeTrend').next();
  var datetime = Utilities.formatDate(
    new Date(), Session.getScriptTimeZone(), 'yyyyMMdd-HHmm'
  );
  folder = folder.createFolder(datetime);
  var categories = YouTube.VideoCategories.list(
    'id,snippet', {regionCode: 'jp'});
  folder.createFile(
    'categories.json', JSON.stringify(categories), 'application/json');
  categories.items.forEach(function(item) {
    var itemNum = 0;
    var pageToken = '';
    while (pageToken != null) {
      try {
        var videos = YouTube.Videos.list('id,snippet', {
          chart: 'mostPopular',
          locale: 'jp',
          regionCode:'jp',
          videoCategoryId: item.id,
          maxResults: 50,
          pageToken: pageToken
        });
        itemNum++;
        folder.createFile(
          'videos-'+item.id+'-'+itemNum+'.json',
          JSON.stringify(videos),
          'application/json'
        );
        pageToken = videos.nextPageToken;
        Utilities.sleep(50);
      } catch (err) {
        Logger.log(
          'category %s not supported (%s)', item.id, err.message);
        pageToken = null;
      }
    }
  });
}
{% endhighlight %}

成功すると、Drive直下にあらかじめ作った`YouTubeTrend`というフォルダの中に、
日付、時刻のフォルダを作成し、その中にトレンドの動画のメタデータを順次格納します。

### 2. Google Colabで集計

以下のスクリプトを実行すると、これまで保存されていたメタデータを集計し、csvファイルに保存します。
こちらもまだ下書き程度ですので、完成したら解説をつけます。

{% highlight python linenos %}
from google.colab import drive
drive.mount('/content/drive')

import os
import json
base_dir = '/content/drive/My Drive/YouTubeTrend/'

ofp = open(os.path.join(base_dir, 'summary.csv'), 'w')
print('time,scatid,rank,id,cid,catid,title,ctitle', file=ofp)

for d, f in [
             (d, f)
             for d in sorted(os.listdir(base_dir))
             if os.path.isdir(os.path.join(base_dir, d))
             for f in os.listdir(os.path.join(base_dir, d))
             ]:
  jsonpath = os.path.join(base_dir, d, f)
  if len(os.path.splitext(f)[0].split('-')) != 3:
    continue
  scatid, batch = tuple(map(int, os.path.splitext(f)[0].split('-')[1:]))
  with open(jsonpath, 'r') as fp:
    jsonobj = json.load(fp)
    for rank_in_batch, item in enumerate(jsonobj['items'], 1):
      # time,scatid,rank,id,cid,catid,title,ctitle
      print(','.join(map(str, (
          d, scatid, (batch-1)*50+rank_in_batch,
          item['id'], item['snippet']['channelId'],
          item['snippet']['categoryId'],
          '"{}"'.format(item['snippet']['title'].replace('"', '""')),
          '"{}"'.format(item['snippet']['channelTitle'].replace('"', '""')),
          ))), file=ofp)
      #print(item['id'])
      #print(item['snippet'].keys())
      #print(item['snippet']['title'])
      #print(item['snippet']['channelTitle'])
      #print(item['snippet']['thumbnails'].keys())
      #print(item['snippet']['channelId'])
      #print(item['snippet']['publishedAt'])
      #print(item['snippet']['categoryId'])
      #print()

ofp.close()
{% endhighlight %}

### 次回予告

一日ほど動かしてみたのですが、目標とする人数(まずは1000人)には程遠いのでしばらく続けます。
それと並行してSyncNetの調査を進めます。

[^1]: http://www.robots.ox.ac.uk/~vgg/data/voxceleb/
