---
title:  "Network Security Perimeter で PaaS のネットワークセキュリティを一元管理する"
date:   2027-02-14 14:34:25
categories: ネットワーク
author_profile: false
tags:
  - Network Security Perimeter
  - Storage Account
  - Key Vault
---

## こんなことを書いています

* Network Security Perimeter (NSP) は PaaS リソースの論理的なネットワーク境界を一元管理できるサービス
* ペリメータ内のリソース同士は暗黙的に通信が可能
* 学習モードと強制モードの違いを検証
* Private Endpoint / サービスエンドポイントとの使い分け

<!--more-->

## Network Security Perimeter とは

PaaS リソースのファイアウォール設定を個別に管理するのは大変です。NSP を利用することで、複数の PaaS リソースのネットワークアクセスを一元的に管理できるようになります。

対応リソースには Storage Account、Key Vault、Event Hub、SQL Database 等があります。

[Network Security Perimeter の概要](https://learn.microsoft.com/ja-jp/azure/private-link/network-security-perimeter-concepts)

## 検証構成

今回の検証構成は以下の通りです。

<!-- TODO: アーキテクチャ図を挿入 -->
<!-- ![architecture](/assets/article_images/2026-02-14-nsp-verification/architecture.png) -->

| リソース | 用途 |
|---|---|
| NSP | ペリメータ本体 |
| Storage Account A | NSP 内リソース |
| Key Vault | NSP 内リソース |
| Storage Account B | NSP 外リソース（比較用） |
| VNet + VM | NSP 外からのアクセステスト用 |

## NSP の作成とリソースの関連付け

### NSP の作成

<!-- TODO: NSP 作成手順のスクリーンショットを挿入 -->

### プロファイルの作成

<!-- TODO: プロファイル作成手順のスクリーンショットを挿入 -->

### リソースアソシエーションの追加

<!-- TODO: リソースアソシエーション追加手順のスクリーンショットを挿入 -->

## アクセスルールの検証

### Inbound ルール

NSP では以下の種類の Inbound ルールを設定できます。

* サブスクリプションベース
* FQDN ベース
* IP アドレスベース

<!-- TODO: 各ルールの設定と動作確認のスクリーンショットを挿入 -->

### Outbound ルール

* FQDN ベース

<!-- TODO: Outbound ルールの設定と動作確認のスクリーンショットを挿入 -->

### ルールなしの場合

<!-- TODO: ルール未設定時のアクセス拒否の確認結果を挿入 -->

## ペリメータ内リソース同士の通信

NSP 内のリソース同士は、明示的なルールを設定しなくても暗黙的に通信が可能です。

<!-- TODO: Storage Account A と Key Vault 間の通信確認結果を挿入 -->

## NSP 外からのアクセス制御

### 学習モード（Learning）

学習モードでは、既存のアクセスパターンを記録しつつ、アクセス自体はブロックしません。

<!-- TODO: 学習モードでのアクセスログ確認結果を挿入 -->

### 強制モード（Enforced）

強制モードに切り替えると、NSP ルール外のアクセスがブロックされます。

<!-- TODO: 強制モードでのアクセスブロック確認結果を挿入 -->

### モード切り替え時の挙動

<!-- TODO: 学習モード → 強制モードに切り替えた際の挙動変化を記載 -->

## 既存の PaaS ファイアウォール設定との関係

各 PaaS リソースにはそれぞれ独自のネットワークアクセス制御設定があります（例：Storage Account の「ファイアウォールと仮想ネットワーク」、Key Vault の「ネットワーク」設定、SQL Database の「ファイアウォール規則」など）。

NSP を適用すると、これらのリソース個別のネットワークアクセス制御より NSP のルールが優先されます。
NSG（VNet/サブネットレベル）や Azure Firewall（VNet トラフィック制御）とはレイヤーが異なるので混同しないように注意が必要です。

<!-- TODO: PaaS ファイアウォール設定との競合検証結果を挿入 -->

## 診断ログの確認

NSP の診断設定を有効にすると、許可/拒否されたアクセスのログを確認できます。

<!-- TODO: 診断ログのスクリーンショットを挿入 -->

## Private Endpoint との比較

| 観点 | NSP | Private Endpoint |
|---|---|---|
| 対象 | 複数の PaaS リソースを一括管理 | リソース単位で設定 |
| ネットワーク | パブリックエンドポイント上での制御 | VNet 内のプライベート IP でアクセス |
| コスト | NSP 自体は無料（プレビュー時点） | Private Endpoint ごとに課金 |
| 管理性 | 一元管理しやすい | リソースごとに管理が必要 |

## まとめ

<!-- TODO: 検証結果のまとめを記載 -->

## Appendix

<!-- TODO: 参考リンクや補足情報を記載 -->
