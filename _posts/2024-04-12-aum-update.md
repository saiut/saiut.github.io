---
title:  "Azure Update Manager のアップデート方法の違い"
date:   2024-04-17 14:34:25
categories: Operation
author_profile: false
tags:
  - "Update Manager"
header:
  overlay_image: /assets/article_images/2024-04-12-aum-update/aum-architecture.png
---

## こんなことを書いています

* Update Manager はパッチ更新管理に便利なサービスです
* パッチ適用のタイミングは様々なパターンがあるので注意

## Azure Update Manager

Windows, Linux 双方とも大事な運用項目の1つとして挙げられるパッチ適用ですが、
Azure では Azure Update Manager（AUM）というサービスを利用して、「どの VM にどのパッチが適用されていないか」などを確認することが可能です。

AUM では、Resource Graph を用いて、VM や Arc 対応サーバーにある拡張機能からパッチの適用状況を引っ張ってきます。

![AUM Architecture](/assets/article_images/2024-04-12-aum-update/aum-architecture.png)

こちらから図を引用しています。：[Azure Update Manager について](https://learn.microsoft.com/ja-jp/azure/update-manager/overview?tabs=azure-vms#key-benefits)

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

その名の通り、ユーザー側で「いつパッチを適用していいか」というタイミングを決めて、そのタイミングでパッチを適用するという方法です。

追加方法は簡単で、AUM 概要から「スケジュールの更新」から設定します。

![Schedule](/assets/article_images/2024-04-12-aum-update/schedule.png)

リソースとして作成していきます。

* メンテナンススコープは「ゲスト(Azure VM、Arc 対応 VM/サーバー)にする
* 「再起動の設定」で、パッチ適用時に再起動してしまっていいか
* 「スケジュール」で、いつから/メンテナンスする期間/期間を決定

![add-Schedule](/assets/article_images/2024-04-12-aum-update/add-schedule.png)

次にどの VM を対象のスケジュールでパッチを当てるかを設定します。

動的スコープを利用することで、今後増えるだろう VM も抜け漏れなく Update Manager でパッチ適用管理が可能になります。

リソースの種類（VM, Arc 対応）やどのリージョンに所属する VM なのかなどを決めましょう。

静的に割り当てる場合は、動的スコープはスルーして、次の「リソース」タブでどの VM かを全て決めます。

![動的スコープ](/assets/article_images/2024-04-12-aum-update/dynamicscope.png)

次に、どのパッチを適用するかを決定します。 KB ID からも決めることが可能です。
![パッチ](/assets/article_images/2024-04-12-aum-update/patch.png)

こういった形で、決められたスケジュールで決められた範囲のパッチを自動で適用できるようになります。

### Azure マネージド

こちらを利用すると、以下に従って Azure 側が必要なパッチを適用します。

* [重大] または [セキュリティ] に分類されているパッチは、自動的にダウンロードされ、VM に適用されます。
* パッチは、IaaS VM のピーク外の時間帯 (VM のタイムゾーン) に適用されます。
* VMSS Flex の場合、パッチはすべての時間に適用されます。
* パッチ オーケストレーションが Azure によって管理され、可用性優先の原則に従ってパッチが適用されます。
* プラットフォーム正常性シグナルによって特定された仮想マシンの正常性は、パッチ適用の失敗を検出するために監視されます。
* アプリケーションの正常性はアプリケーション正常性拡張機能を使って監視できます。
* すべての VM サイズで機能します。

こちらからの引用です。
[Azure VM での VM ゲストの自動パッチ適用](https://learn.microsoft.com/ja-jp/azure/virtual-machines/automatic-vm-guest-patching)

その中で「可用性優先の原則」と記載がありますが、パッチ適用によって可用性が下がらないように担保するようにパッチ適用するよ、というものです。
以下のような原則があります。（あくまで一部ですがわかりやすいものを記載）

* geo ペアリージョン（東日本と西日本みたいなペア）同時に更新しない
* 異なる AZ にある VM が同時に更新されない
* 共通の AS 内にある VM が同時に更新されない

また、Azure マネージドの場合はパッチオーケストレーションモードが __AutomaticByPlatform__ である必要があります。

仮想マシン作成時の「管理」タブにある「ゲスト OS の更新プログラム」で選択可能です。（Azure-orchestrated using Automatic guest patching）
![AutomaticByPlatform](/assets/article_images/2024-04-12-aum-update/automaticbyplatform.png)

また、以下のコマンドによって、対象 VM のパッチオーケストレーションモードを確認することが可能です。後から出てくるモードも同じく確認可能です。

```bash
 az vm get-instance-view --resource-group <rgname> --name <vmname>
 ```

![CLI](/assets/article_images/2024-04-12-aum-update/cli-check.png)

### Windows 自動更新/イメージの既定値

こちらは非常に簡単で、OS 側の設定に任せる、という設定です。

* Windows 自動更新： Windows OS のみ
* イメージの既定値： Linux OS のみ

### 手動更新

## まとめ

