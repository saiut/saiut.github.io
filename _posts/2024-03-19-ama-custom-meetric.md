---
title:  "Azure VM のゲスト OS 上のパフォーマンスカウンターを気軽に取ってみる"
date:   2024-03-19 05:34:25
categories: Monitor
author_profile: false
tags:
  - AMA
  - "メトリック"
---

## TL;DR

* データ収集ルールを使えば簡単に Azure VM 上のゲストメトリックが取得可能
* メトリックを Log Analytics Workspace に入れるまでもないときに活用

## カスタムメトリックの取得

Azure VM では、仮想マシンの基盤から VM のメトリックを自動で取得してくれます。

ただ、あくまで基盤のメトリックなので VM 内部のメトリックは取得できません。カスタムメトリックを利用することで、ゲスト OS のメトリック（例えば Windows のパフォーマンスカウンター）を取得し、メトリックスエクスプローラーで確認が可能となります。

昔はもうちょっと面倒だったなーという記憶があるのですが、 Azure Monitor Agent がリリースされてデータ収集ルール（DCR）が使える現状では、
DCR を活用すれば簡単に取得することが可能です。

> {: .notice--primary}
> AMA を利用したカスタムメトリックはプレビュー機能になります。

----

## DCR からゲスト OS のメトリックを取得する

Azure Monitor の DCR を活用することで、DCR で設定した収集するデータ、データを収集する VM、送信先を簡単に管理することが可能です。

![DCR Data Source](/assets/article_images/2024-03-19-ama-custom-metric/dcr-datasource.png)

どのデータを取得するか

![DCR Resource](/assets/article_images/2024-03-19-ama-custom-metric/dcr-resource.png)

どの VM から設定したデータを取得するか


データソースについては、デフォルトではベーシックなものが選ばれています。
![DCR Resource](/assets/article_images/2024-03-19-ama-custom-metric/dcr-performancecounter.png)

カスタムにすることで、細かく取得が可能になります。
![DCR Resource Custom](/assets/article_images/2024-03-19-ama-custom-metric/dcr-performancecounter-custom.png)

## パフォーマンスカウンターは _total しかデフォルトでとれないので各プロセスのカウンターを取得する

カスタムで取得できるのは基本的にそれぞれのパフォーマンスで「_total」や「*」といった、全プロセスなどを足し合わせたものを取得するので、
例えば特定プロセスの CPU 使用率を取得したい場合はパフォーマンスカウンターを追加してあげる必要があります。

```text
\Process(プロセス名)\ % Processor Time
```

取得したいプロセス名を全部いれる必要があるので手間ではありますが。。。

![DCR Add Performance](/assets/article_images/2024-03-19-ama-custom-metric/dcr-add-performance.png)

設定してあげることで、VM のメトリック名前空間に「仮想マシンのゲスト」が追加され、そこから DCR で設定したパフォーマンスカウンターのメトリックを確認することが可能です。

こちらの例では、Azure の拡張機能である Network Watcher Agent のスレッド数を取得しています。

![Custom Metric](/assets/article_images/2024-03-19-ama-custom-metric/custommetric.png)

## Appendix

[Azure Monitor のカスタム メトリック (プレビュー)](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/metrics-custom-overview)