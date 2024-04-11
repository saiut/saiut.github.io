---
title:  "Azure Site Recovery でフェールオーバーしたときは VM 拡張機能 に気を付ける"
date:   2024-04-14 14:34:25
categories: DR
author_profile: false
tags:
  - ASR
---

## こんなことを書いています

* ASR のフェールオーバーは拡張機能の移行をサポートしていない
* AMA でログ取得している場合、フェールオーバー後はログが取得されなくなってしまう
* AMA の DCR を活用して、VM の再登録などを復旧計画に入れてしまうことを忘れずに

## ASR の制限事項をおさらいする

Azure Site Recovery の制限事項として、「拡張機能は再インストールしてください」というものがあります。

[Azure リージョン間での Azure VM ディザスター リカバリーに関するサポート マトリックス](https://learn.microsoft.com/ja-jp/azure/site-recovery/azure-to-azure-support-matrix#replicated-machines---compute-settings)

![Limitation](/assets/article_images/2023-10-07-asr-extension/limitation.jpg)

拡張機能は、公式サイトから引用すると以下のようなもので、インストールすることで Azure VM に様々な機能を提供することが可能になっています。
> 拡張機能は、Azure 仮想マシン (VMs)でのデプロイ後の構成と自動化を提供する小さなアプリケーションです。 Azure プラットフォームでは、VM の構成、監視、セキュリティ、およびユーティリティのアプリケーションを対象とする多くの拡張機能をホストします。

[Azure 仮想マシンの拡張機能とその機能](https://learn.microsoft.com/ja-jp/azure/virtual-machines/extensions/overview)

代表的な拡張機能として Azure VM でログを取得する Azure Monitor Agent （AMA）がありますが、ASR でフェールオーバーした後は拡張機能を再度インストールしない限り、AMA を利用してログを取得することができません。

----

## 拡張機能を再インストールする

ここでは AMA を例にして実際に ASR を使ってフェールオーバーした VM に対して AMA を再インストールする流れを見てみます。

VM をデプロイし、AMA でログを取得した場合、VM の拡張機能はこのようになっています。

![VM 拡張機能](/assets/article_images/2023-10-07-asr-extension/vmextension.jpg)

対象の VM をフェールオーバーした後、改めて VM の拡張機能を確認してみると、サポートされていないためすべてなくなっています。
（対象の VM 名が一緒なのでわかりにくいですが）

![フェールオーバー後](/assets/article_images/2023-10-07-asr-extension/vmextension_afterfailover.jpg)

一応コマンドで確認すると…

```text
 Type                : Microsoft.Azure.Monitor.AzureMonitorWindowsAgent
    TypeHandlerVersion  : 1.20.0.0
    Status              :
      Code              : ProvisioningState/Unresponsive
      Level             : Warning
      DisplayStatus     : Unresponsive
      Message           : Handler Microsoft.Azure.Monitor.AzureMonitorWindowsAgent of version 1.20.0.0 is unresponsive. Last
heartbeat: 10/6/2023 5:50:06 AM
```

「Unresponsive」と書かれていて、AMA の反応がないことがわかります。
この場合、AMA でログ取得するデータ収集ルールにおいても、フェールオーバーを行った VM を選択できません。
ですので、一度 AMA 拡張機能をアンインストールする必要があります。

コマンド

```powershell
Remove-AzVMExtension -Name AzureMonitorWindowsAgent -ResourceGroupName <resource-group-name> -VMName <virtual-machine-name>
```

結果

```text
Virtual machine extension removal operation
This cmdlet will remove the specified virtual machine extension. Do you want to continue?
[Y] Yes  [N] No  [S] Suspend  [?] Help (default is "Y"): Y
RequestId IsSuccessStatusCode StatusCode ReasonPhrase
                         True  NoContent No Content
```

削除をしてあげることで、データ収集ルールで VM を選択できるようになっているので追加してあげます。

![データ収集ルール](/assets/article_images/2023-10-07-asr-extension/dcr.jpg)

これでログが無事取得できるようになりました。

ただ、この方法だと人力すぎますよね。

## 復旧計画で AMA 再インストールを自動化する

ASR では復旧計画というものがあり、DR 発動時に行う作業を1つの流れにまとめることが可能です。

例えば、復旧する順番として「DB サーバーを先に起動した後に Web サーバーを起動する」といった流れですね。 FO 時に起動の順番を決定することができるのが、復旧計画です。
この復旧計画を活用し、 ASR 発動後に AMA を再インストールするという流れにしてあげることで、自動で AMA が再インストールされてログが取れるようになります。

## Automation でフェールオーバー対象の VM の拡張機能を再インストールしてしまう

復旧計画では、Azure Automation のスクリプトを利用することが可能です。
