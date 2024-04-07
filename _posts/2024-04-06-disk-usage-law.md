---
title:  "Azure VM のディスク使用量を取得しよう"
date:   2024-04-06 14:34:25
categories: Monitor
author_profile: false
tags:
  - Storage
  - "Log Analytics"
  - "ログ"
  - "メトリック"
---

## こんなことを書いています

* Azure のホストメトリックではディスク使用量/率は取得できない
* ディスク使用量はカスタムメトリックか Log Analytics にログとして送って取得しよう

## VM のディスク使用量

ディスクの IOPS や Read/Write といったディスクのパフォーマンスについては取得できるのですが、ディスク使用量は現状 VM のホストメトリックからは取得することができません。

ちなみに Azure 基盤側から VM のディスクの使用量まで取れないことに起因しています。

監視としてディスクの使用量が少なくなったなどは実施することは多いと思いますので、取得するようにしていきましょう。

> {: .notice--primary}
> VM ディスクのパフォーマンスをどう考えたらいいのか？については以下が非常に参考になります。
>
> [Azure VM のディスクパフォーマンス状況の確認方法について](https://jpaztech.github.io/blog/vm/disk-metrics/)

2024/4現在、ディスクの使用量（空き容量）を取得するには、ゲスト VM 側の情報を取得する必要があります。

## カスタムメトリックを利用する

Azure 基盤側からみたメトリックではなく、ゲスト OS 側で取得できる「カスタムメトリック」を利用することで、ディスクの使用量を取得することが可能です。
別記事でカスタムメトリックを取得する方法を記載していますので、こちらを参考にしてカスタムメトリックを取得してください。

[Azure VM のゲスト OS 上のパフォーマンスカウンターを気軽に取ってみる](/_posts/2024-03-19-ama-custom-meetric.md)

Linux の場合は、「disk/used_percent」などで取得するといいでしょう。

![Linux Free Disk Usage](/assets/article_images/2024-04-08-disk-usage-law/linux-custom-metric.png)

Windows の場合は使用率ではなく、使用量なので「Free Space」を取得するといいかと。
（こちらの例では_toralで取得しており、全ディスクの空き容量を取得してしまっているので、それぞれのディスクを分けて取得したい場合は C,D,E など分けてカスタムメトリックを取得する必要があります）

![Windows Free Space](/assets/article_images/2024-04-08-disk-usage-law/windows-custom-metric.png)

## Log Analytics を利用する

もう1つの方法は Log Analytics に対してパフォーマンスカウンターなどのメトリックを飛ばし、Log Analytics からクエリを抽出する方法です。
カスタムメトリックを取得する方法と同じように、DCR を利用することでパフォーマンスカウンターを Log Analytics にデータを送信することが可能です。

[Azure VM のゲスト OS 上のパフォーマンスカウンターを気軽に取ってみる](/_posts/2024-03-19-ama-custom-meetric.md)

ゲスト OS のパフォーマンスデータを Log Analytics に送信できたら、このような形でクエリを打ちこんでみましょう

```Powershell
# Linux の場合
Perf
 | where CounterName == "% Used Space"
 | where InstanceName == "/"
 | where Computer == ">VM Name>"
# Windows の場合
Perf
| where Computer == "<VM Name>"
| where CounterName == "% Free Space"
```

![Linux Disk Law](/assets/article_images/2024-04-08-disk-usage-law/linux-law.png)

![Windows Disk Law](/assets/article_images/2024-04-08-disk-usage-law/windows-law.png)

ディスクの使用量が取得できるはずです。

## アラートを設定したい場合

### Log Analytics の場合

CounterValue で使用率が 80% を超えたら、などになると思いますので、

```Powershell
CounterValue >= 80
```

といったものを追加してあげて、アラートを設定してあげるといいでしょう。

### カスタムメトリックの場合

特に考えることなく、対象のカスタムメトリックからアラートを作成してあげましょう。

![Metric Alert](/assets/article_images/2024-04-08-disk-usage-law/metric-alert.png)