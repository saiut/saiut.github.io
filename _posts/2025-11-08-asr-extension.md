---
title:  "Azure Site Recovery でフェールオーバーしたときは VM 拡張機能を再インストールすることを忘れずに"
date:   2025-11-08 14:34:25
categories: DR
author_profile: false
tags:
  - Azure Site Recovery
  - Azure Monitor Agent
header:
  teaser: /assets/article_images/2025-11-08-asr-extension/vmextension.jpg
---

## こんなことを書いています

* ASR のフェールオーバーは拡張機能の移行をサポートしていない
* AMA でログ取得している場合、フェールオーバー後はログが取得されなくなってしまう
* AMA の DCR を活用して、VM の再登録などを復旧計画に入れてしまうことを忘れずに

<!--more-->

## ASR の制限事項

Azure Site Recovery の制限事項の 1 つとして、「拡張機能は再インストールしてください」というものがあります。

[Azure リージョン間での Azure VM ディザスター リカバリーに関するサポート マトリックス](https://learn.microsoft.com/ja-jp/azure/site-recovery/azure-to-azure-support-matrix#replicated-machines---compute-settings)

![Limitation](/assets/article_images/2025-11-08-asr-extension/limitation.jpg)

そもそも拡張機能は公式サイトから引用すると以下のようなもので、「Azure VM に様々な機能を提供する軽量なアプリケーション」です。
> 拡張機能は、Azure 仮想マシン (VMs)でのデプロイ後の構成と自動化を提供する小さなアプリケーションです。 Azure プラットフォームでは、VM の構成、監視、セキュリティ、およびユーティリティのアプリケーションを対象とする多くの拡張機能をホストします。

[Azure 仮想マシンの拡張機能とその機能](https://learn.microsoft.com/ja-jp/azure/virtual-machines/extensions/overview)

代表的な拡張機能として Azure VM でログを取得する Azure Monitor Agent （AMA）がありますが、ASR でフェールオーバーした後は制限事項にある通り、拡張機能を再インストールしない限り、AMA を利用してログを取得することができません。

----

## 拡張機能を再インストールする

ここでは AMA を例にして実際に ASR を使ってフェールオーバーした VM に対して AMA を再インストールする流れを見てみます。

VM をデプロイし、AMA でログを取得した場合、VM の拡張機能はこのようになっています。（AMA 以外の拡張機能も含まれています、AMA は AzureMonitorWindowsAgent です）

![VM 拡張機能](/assets/article_images/2025-11-08-asr-extension/vmextension.jpg)

対象の VM をフェールオーバーした後、改めて VM の拡張機能を確認してみると、サポートされていないためすべてなくなっています。
（対象の VM 名が一緒なのでわかりにくいですが）

![フェールオーバー後](/assets/article_images/2025-11-08-asr-extension/vmextension_afterfailover.jpg)

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
（下記のスクリーンショットにはないのですが、AMA 拡張機能削除前は対象の VM は選べない状態となっています）

![データ収集ルール](/assets/article_images/2025-11-08-asr-extension/dcr.jpg)

これでログが無事取得できるようになりました。

ただ、この方法だと人力すぎますよね。

## 復旧計画で AMA 再インストールを自動化する

ASR では復旧計画というものがあり、DR 発動時に行う作業を1つの流れにまとめることが可能です。

例えば、復旧する順番として「DB サーバーを先に起動した後に Web サーバーを起動する」といった流れのように、 FO 時に起動の順番を決定することができるのが、復旧計画です。
この復旧計画を活用し、 ASR 発動後に AMA を再インストールするという流れにしてあげることで、自動で AMA が再インストールされてログが取れるようになります。

## Automation でフェールオーバー対象の VM の拡張機能を再インストールしてしまう

復旧計画では、Azure Automation のスクリプトを利用することが可能です。
流れとしては以下のような流れになるでしょう。

1. FO が開始され、別リージョンで VM が起動する
2. 別リージョンの VM から AMA をアンインストールするコマンドを実行する -> Automation で実行
3. AMA をインストールするコマンドが実行される
4. データ収集ルール（DCR）に VM を再関連付けする

以下は、Azure Automation の Runbook で使用できる AMA のインストールスクリプトの一例です。

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$vmName,
    
    [Parameter(Mandatory=$true)]
    [string]$resourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$location,
    
    [Parameter(Mandatory=$true)]
    [string]$dcrId
)

# Azure に接続（Managed Identity を使用）
Connect-AzAccount -Identity

# 既存の AMA 拡張機能を削除（エラーハンドリング付き）
try {
    Write-Output "Removing existing AMA extension from VM: $vmName"
    Remove-AzVMExtension -Name "AzureMonitorWindowsAgent" -ResourceGroupName $resourceGroupName -VMName $vmName -Force
    Write-Output "AMA extension removed successfully"
} catch {
    Write-Output "Failed to remove AMA extension or extension not found: $($_.Exception.Message)"
}

# AMA 拡張機能をインストール
try {
    Write-Output "Installing AMA extension on VM: $vmName"
    Set-AzVMExtension -ResourceGroupName $resourceGroupName -VMName $vmName -Name "AzureMonitorWindowsAgent" -Publisher "Microsoft.Azure.Monitor" -Type "AzureMonitorWindowsAgent" -TypeHandlerVersion "1.0" -Location $location
    Write-Output "AMA extension installed successfully"
} catch {
    Write-Error "Failed to install AMA extension: $($_.Exception.Message)"
    throw
}

# データ収集ルールとの関連付け
try {
    Write-Output "Associating VM with Data Collection Rule: $dcrId"
    $vm = Get-AzVM -ResourceGroupName $resourceGroupName -Name $vmName
    $associationName = "vm-$vmName-dcr-association"
    New-AzDataCollectionRuleAssociation -ResourceUri $vm.Id -RuleName $associationName -DataCollectionRuleId $dcrId
    Write-Output "DCR association completed successfully"
} catch {
    Write-Error "Failed to associate VM with DCR: $($_.Exception.Message)"
    throw
}

Write-Output "AMA reinstallation and DCR association completed for VM: $vmName"
```

## 復旧計画での実装手順

1. **Azure Automation アカウントの準備**
   - Automation アカウントを作成
   - 必要な PowerShell モジュール（Az.Accounts、Az.Monitor など）をインポート
   - システム割り当て Managed Identity を有効化し、適切な権限を付与

2. **Runbook の作成**
   - 上記のスクリプトを Runbook として作成
   - パラメータを適切に設定

3. **復旧計画への組み込み**
   - ASR の復旧計画でグループを作成
   - フェールオーバー後のアクションとして Automation Runbook を追加
   - 必要なパラメータ（VM名、リソースグループ、DCR ID など）を設定

## 注意事項

### Linux VM の場合

Linux VM の場合は拡張機能の名前が異なるため、以下のように修正が必要です。

```powershell
# Linux の場合
Remove-AzVMExtension -Name "AzureMonitorLinuxAgent" -ResourceGroupName $resourceGroupName -VMName $vmName -Force
Set-AzVMExtension -ResourceGroupName $resourceGroupName -VMName $vmName -Name "AzureMonitorLinuxAgent" -Publisher "Microsoft.Azure.Monitor" -Type "AzureMonitorLinuxAgent" -TypeHandlerVersion "1.0" -Location $location
```

### 複数の拡張機能がある場合

AMA 以外にも拡張機能を利用している場合は、それらも同様に再インストールが必要です。例えば：

- Dependency Agent
- カスタム拡張機能

これらも同様の手順で復旧計画に組み込むことができます。

## まとめ

ASR を利用する際は、拡張機能の制限事項を理解し、事前に復旧計画に組み込んでおくことが重要です。特に監視やセキュリティに関わる拡張機能は、DR 発動後も継続して動作させる必要があるため、自動化しておくことをお勧めします。

復旧計画と Azure Automation を活用することで、手動での作業を減らし、確実な復旧を実現することができます。定期的な DR テストの際に、拡張機能の動作確認も含めて実施することを忘れずに行いましょう。