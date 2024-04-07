---
title:  "Azure Update Manager のアップデート方法の違い"
date:   2024-04-12 14:34:25
categories: Operation
author_profile: false
tags:
  - "Update Manager"
---

## こんなことを書いています

* Update Manager は WSUS の置き換えではないです
* パッチ適用のタイミングは様々なパターンがあるので注意

## Azure Update Manager

Windows, Linux 双方とも大事な運用項目の1つとして挙げられるパッチ適用ですが、
Azure では Azure Update Manager（AUM）というサービスを利用して、「どの VM にどのパッチが適用されていないか」などを確認することが可能です。

AUM では、 Resource Graph を用いて、VM や Arc 対応サーバーにある拡張機能からパッチの適用状況を引っ張ってきます。

![AUM Architecture](/assets/article_images/2024-04-12-aum-update/aum-architecture.png)

こちらから図を引っ張ってきています

[Azure Update Manager について](https://learn.microsoft.com/ja-jp/azure/update-manager/overview?tabs=azure-vms#key-benefits)

パッチ適用状態の確認はオンデマンド/定期的に実行され、さらにパッチ適用もオンデマンド/定期的に実行可能です。

このうち、パッチを適用する「定期的な」タイミングが様々存在しているので、そちらをまとめたいと思います。

## パッチ適用タイミングの色々

大きく分けて 4 つあります。

| パッチオーケストレーション | どのような挙動か |
| --- | --- |
| Customer Managed Schedules | ユーザーが決めたスケジュールでパッチを適用する |
| Azure マネージド - 安全なデプロイ | Azure 側で自動的にパッチを適用する |
| Windows 自動更新/イメージの既定値 | ゲスト OS 側の設定に任せる |
| 手動更新 | Windows OS 側も自動更新が無効になるため、自身で更新 |

### Customer Managed Schedules



### Azure マネージド

### Windows 自動更新/イメージの既定値

### 手動更新