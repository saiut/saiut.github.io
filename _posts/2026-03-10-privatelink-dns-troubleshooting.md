---
title:  "Private Link + Private DNS Zone の「あるある」トラブルシューティング集"
date:   2026-03-10 09:00:00
categories: ネットワーク
author_profile: false
tags:
  - Azure Private Link
  - Azure Private Endpoint
  - Azure Private DNS Zone
  - DNS
  - トラブルシューティング
header:
  overlay_image: /assets/article_images/2026-03-10-privatelink-dns-troubleshooting/eyecatch.svg
  teaser: /assets/article_images/2026-03-10-privatelink-dns-troubleshooting/eyecatch.svg
---

## こんなことを書いています

* Azure Private Endpoint を使った際に名前解決がうまくいかない「あるある」パターン 7 選
* 各パターンの事象・原因・解決策を具体的に整理
* nslookup / Resolve-DnsName などの確認コマンド例付き
* Hub-Spoke 構成やオンプレミス接続時に特にハマりやすいポイントを重点解説

<!--more-->

## はじめに

Azure Private Endpoint を使うと、Azure PaaS サービスへの通信を VNet 内のプライベート IP 経由に閉じることができます。

ただし、Private Endpoint を作成しただけでは十分ではありません。**DNS の名前解決が正しく Private IP を返すように構成する**必要があります。

そこで重要になるのが Azure Private DNS Zone です。Private DNS Zone にレコードを用意し、対象の VNet にリンクすることで、VNet 内の VM や各種サービスが Private Endpoint の Private IP を名前解決できるようになります。

仕組み自体はシンプルですが、実務では「名前解決できない」「なぜかパブリック IP に解決される」「オンプレミスからだけつながらない」といったトラブルがよく発生します。

本記事では、特によくある 7 つのパターンを取り上げ、事象、原因、解決策の順で整理します。

## パターン 1: Private DNS Zone を VNet にリンクし忘れて名前解決できない

### 事象

Private Endpoint を作成し、Private DNS Zone にも A レコードが登録されているのに、VM から `nslookup` するとパブリック IP が返ってくる。

```
> nslookup myaccount.blob.core.windows.net
Non-authoritative answer:
Name:    blob.xxxx.store.core.windows.net
Address:  20.xx.xx.xx   ← パブリック IP
```

### 原因

Private DNS Zone を作成しただけでは、VNet 内のリソースはそのゾーンを参照しません。**Private DNS Zone の「仮想ネットワーク リンク」が未設定**のため、VNet の DNS 解決に Private DNS Zone が使われていない状態です。

### 解決策

Azure Portal で Private DNS Zone の **[仮想ネットワーク リンク]** を開き、対象の VNet をリンクします。

```powershell
# PowerShell でリンクを作成する例
New-AzPrivateDnsVirtualNetworkLink `
  -ResourceGroupName "rg-network" `
  -ZoneName "privatelink.blob.core.windows.net" `
  -Name "link-to-vnet-spoke01" `
  -VirtualNetworkId "/subscriptions/<sub-id>/resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-spoke01" `
  -EnableRegistration $false
```

リンク作成後、再度名前解決を確認します。

```
> nslookup myaccount.blob.core.windows.net
Name:    myaccount.privatelink.blob.core.windows.net
Address:  10.0.1.4   ← Private IP
```

### ポイント

- **`-EnableRegistration` は `$false`** にしてください。`$true` にすると VM のホスト名を Private DNS Zone に自動登録する機能が有効になりますが、PaaS の Private Endpoint の名前解決では通常不要です。
- 1 つの Private DNS Zone に対して複数の VNet をリンクできます。

## パターン 2: オンプレミスや他 VNet から Private Endpoint の名前解決ができない

### 事象

Private Endpoint を配置した VNet 内の VM からは名前解決できるが、VPN/ExpressRoute 経由のオンプレミスや、ピアリングしている別の VNet からは名前解決できない。

### 原因

Azure Private DNS Zone は、基本的に **リンクされた VNet 内** でのみ有効です。オンプレミスの DNS サーバーは Azure Private DNS Zone を直接参照できません。

そのため、オンプレミスのクライアントはパブリック DNS を引いてしまい、結果としてパブリック IP が返ります。

### 解決策

**DNS フォワーダー（条件付きフォワーダー）** を構成し、`privatelink.*.core.windows.net` などのクエリを Azure 側へ転送できるようにします。

#### 方法 A: Azure DNS Private Resolver を使う（推奨）

Azure DNS Private Resolver を Hub VNet にデプロイし、インバウンドエンドポイントの IP をオンプレミス DNS サーバーの条件付きフォワーダー先に設定します。

```
オンプレ DNS サーバー
  → 条件付きフォワーダー: privatelink.blob.core.windows.net → 10.0.0.4 (DNS Private Resolver のインバウンド IP)
    → Azure DNS (168.63.129.16)
      → Private DNS Zone の A レコード → Private IP
```

#### 方法 B: Azure VM 上に DNS フォワーダーを構成する

Hub VNet 上の VM に DNS サーバー（Windows DNS や Unbound など）を構成し、`168.63.129.16` への条件付きフォワーダーを設定します。オンプレミスの DNS サーバーからはこの VM の IP にフォワードします。

### ポイント

- Azure DNS Private Resolver は VM 不要のマネージドサービスなので、新規構築では方法 A を推奨します。
- オンプレミス側のフォワーダー設定では、`privatelink` を含む FQDN のゾーンを対象にしてください。

## パターン 3: Private DNS Zone のゾーン名を間違えている

### 事象

Private DNS Zone を作成し、VNet リンクも設定しているが、名前解決が Private IP にならない。

### 原因

Azure サービスごとに **Private DNS Zone のゾーン名が異なります**。間違ったゾーン名で作成すると、DNS クエリがそのゾーンにマッチしないため、パブリック DNS にフォールバックします。

### よくある間違いの例

| サービス | 正しいゾーン名 | ありがちな間違い |
|---|---|---|
| Blob Storage | `privatelink.blob.core.windows.net` | `privatelink.storage.core.windows.net` |
| Azure SQL Database | `privatelink.database.windows.net` | `privatelink.sql.azure.com` |
| Key Vault | `privatelink.vaultcore.azure.net` | `privatelink.keyvault.azure.net` |
| Azure Cosmos DB (SQL API) | `privatelink.documents.azure.com` | `privatelink.cosmosdb.azure.com` |
| Azure Monitor (Log Analytics) | `privatelink.oms.opinsights.azure.com` など複数 | ゾーンの一部が欠落 |

### 解決策

Microsoft Learn の公式ドキュメント [Azure プライベート エンドポイントの DNS 構成](https://learn.microsoft.com/ja-jp/azure/private-link/private-endpoint-dns) に、サービスごとの正しいゾーン名一覧が掲載されています。必ずこのドキュメントを参照してください。

### ポイント

- 特に **Azure Monitor 関連** は、利用する機能に応じて `privatelink.oms.opinsights.azure.com`、`privatelink.ods.opinsights.azure.com`、`privatelink.agentsvc.azure-automation.net`、`privatelink.monitor.azure.com`、`privatelink.blob.core.windows.net` など複数のゾーンが必要になりやすく、漏れやすいです。
- Private Endpoint 作成時の **自動 DNS 統合**（後述のパターン 7）を使えば、正しいゾーン名で自動作成されるため、ゾーン名の間違いを防げます。

## パターン 4: パブリック DNS が優先されて Private IP に解決されない

### 事象

VNet 内の VM から名前解決しても、Private IP ではなくパブリック IP が返ってくる。Private DNS Zone のリンクは設定済み。

### 原因

VNet または VM の **DNS 設定がカスタム DNS サーバーを指している** 場合、Azure 既定の DNS（168.63.129.16）は使われません。

そのカスタム DNS サーバーが `privatelink.*` のクエリを適切に処理できていないと、Private DNS Zone が参照されず、パブリック DNS にフォールバックしてしまいます。

### 確認方法

```powershell
# VM の DNS 設定を確認
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"

# VNet の DNS 設定を Azure CLI で確認
az network vnet show --name vnet-spoke01 --resource-group rg-network --query "dhcpOptions.dnsServers"
```

### 解決策

カスタム DNS サーバー（例: Hub VNet 上の DC や DNS サーバー）に、`privatelink.*` ゾーンの**条件付きフォワーダー**を `168.63.129.16` 宛に設定してください。

Windows DNS サーバーの場合:

```powershell
# 条件付きフォワーダーの追加例
Add-DnsServerConditionalForwarderZone `
  -Name "privatelink.blob.core.windows.net" `
  -MasterServers 168.63.129.16
```

### ポイント

- `168.63.129.16` は Azure のワイヤサーバーで、**Azure VM 内からのみ到達可能**です。オンプレミスの DNS サーバーから直接フォワードすることはできません（パターン 2 参照）。
- VNet の DNS 設定を「既定（Azure 提供）」に戻すと解消するケースもありますが、Active Directory 環境ではカスタム DNS が必要なことが多いため、条件付きフォワーダーで対応するのが現実的です。

## パターン 5: Hub-Spoke 構成で Private DNS Zone のリンク先 VNet を間違える

### 事象

Hub-Spoke 構成で、Spoke VNet 上の VM から Private Endpoint の名前解決ができない。Hub VNet 上の VM からは名前解決できる。

### 原因

Private DNS Zone の仮想ネットワーク リンクが **Hub VNet のみに設定**されており、Spoke VNet にリンクされていないケースです。

Hub-Spoke 構成では、DNS の設計パターンによって対応が異なります。

### 解決策

#### パターン A: Spoke VNet が Azure 既定 DNS を使っている場合

Spoke VNet にも Private DNS Zone の仮想ネットワーク リンクを追加します。

```powershell
# Spoke VNet へのリンク追加
New-AzPrivateDnsVirtualNetworkLink `
  -ResourceGroupName "rg-network" `
  -ZoneName "privatelink.blob.core.windows.net" `
  -Name "link-to-vnet-spoke01" `
  -VirtualNetworkId "/subscriptions/<sub-id>/resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-spoke01"
```

#### パターン B: Spoke VNet のカスタム DNS が Hub の DNS サーバーを指している場合

Hub VNet のリンクのみで OK ですが、Hub 上の DNS サーバー（またはDNS Private Resolver）が `168.63.129.16` に条件付きフォワードしている必要があります。

```
Spoke VM → Hub DNS サーバー → 168.63.129.16 → Private DNS Zone → Private IP
```

この設計では、**Hub 側に DNS を集約し、Spoke が常に Hub の DNS を利用する** 前提であれば、実装上は Private DNS Zone を **Hub VNet のみにリンク**して名前解決できるケースがあります。ただし、Microsoft Learn では、名前解決が必要な VNet をそれぞれ Private DNS Zone にリンクする構成が基本です。中央集約型 DNS 設計を採る場合は、この前提を明示して設計してください。

### ポイント

- Hub-Spoke が多数ある環境では、**パターン B の中央集約型 DNS 設計**は管理しやすい一方で、Spoke が Hub DNS を必ず利用することを前提に設計・運用する必要があります。
- Azure DNS Private Resolver を Hub に配置し、Spoke VNet の DNS 設定でその IP を利用する構成は、実務でも採用しやすいパターンです。

## パターン 6: DNS は解決できるのにアプリから接続できない

### 事象

`nslookup` や `Resolve-DnsName` では Private IP が正しく返ってくるのに、アプリケーション（Web アプリ、SDK など）から Private Endpoint 経由で接続できない。

```powershell
# DNS は正しく解決できている
Resolve-DnsName myaccount.blob.core.windows.net
# → 10.0.1.4 (Private IP) ✓

# しかし接続するとタイムアウトする
Test-NetConnection -ComputerName myaccount.blob.core.windows.net -Port 443
# → TcpTestSucceeded : False
```

### 原因（複数の可能性）

#### 原因 A: NSG / ファイアウォールでブロックされている

Private Endpoint のサブネットに関連付けられた NSG や、Azure Firewall / NVA がトラフィックをブロックしている場合があります。

```powershell
# Private Endpoint への疎通確認
Test-NetConnection -ComputerName 10.0.1.4 -Port 443
```

#### 原因 B: プロキシ設定

アプリケーションや OS にプロキシが設定されていると、DNS 解決はローカルで行われても、実際の HTTP 通信がプロキシ経由になり、プロキシサーバー側でパブリック IP に解決されてしまうことがあります。

```powershell
# プロキシ設定の確認
netsh winhttp show proxy

# 環境変数の確認
$env:HTTP_PROXY
$env:HTTPS_PROXY
```

#### 原因 C: Split-brain DNS

`nslookup` はシステムの DNS キャッシュやホストファイルを無視して直接 DNS サーバーに問い合わせます。一方、アプリケーションは OS の名前解決を使います。`hosts` ファイルや DNS クライアントキャッシュに古いエントリが残っていると、アプリだけ異なる IP に解決される場合があります。

```powershell
# DNS キャッシュの確認
Get-DnsClientCache | Where-Object { $_.Entry -like "*myaccount*" }

# hosts ファイルの確認
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String "myaccount"

# DNS キャッシュのクリア
Clear-DnsClientCache
```

### ポイント

- `nslookup` の結果だけで「DNS は問題ない」と判断しないことが重要です。`Resolve-DnsName`、`dig`、アプリケーション実行時の挙動を合わせて確認し、どこで結果が変わるのかを切り分けましょう。
- プロキシが原因の場合は、Private Endpoint の FQDN をプロキシのバイパスリストに追加します。

## パターン 7: 自動 DNS 統合を使わず A レコードが登録されていない

### 事象

Private Endpoint を作成したが、Private DNS Zone に A レコードが存在せず、名前解決が Private IP にならない。

### 原因

Azure Portal で Private Endpoint を作成する際、**「プライベート DNS ゾーンとの統合」** オプションがあります。このオプションを「いいえ」にした場合、または ARM テンプレート / Terraform / Bicep で `privateDnsZoneGroups` を構成しなかった場合、A レコードは自動登録されません。

### 解決策

#### 方法 A: Private DNS Zone Group を後から追加する（推奨）

```powershell
# Private DNS Zone Group の作成
$zone = Get-AzPrivateDnsZone -ResourceGroupName "rg-network" -Name "privatelink.blob.core.windows.net"

$config = New-AzPrivateDnsZoneConfig `
  -Name "privatelink-blob-core-windows-net" `
  -PrivateDnsZoneId $zone.ResourceId

New-AzPrivateDnsZoneGroup `
  -ResourceGroupName "rg-app" `
  -PrivateEndpointName "pe-storage01" `
  -Name "default" `
  -PrivateDnsZoneConfig $config
```

Private DNS Zone Group を構成すると、A レコードが自動的に作成・管理されます。Private Endpoint の IP が変わった場合も自動更新されるため、**この方法を強く推奨**します。

#### 方法 B: 手動で A レコードを追加する

```powershell
# 手動で A レコードを登録する例
New-AzPrivateDnsRecordSet `
  -ResourceGroupName "rg-network" `
  -ZoneName "privatelink.blob.core.windows.net" `
  -Name "myaccount" `
  -RecordType A `
  -Ttl 3600 `
  -PrivateDnsRecords (New-AzPrivateDnsRecordConfig -IPv4Address "10.0.1.4")
```

### ポイント

- 手動登録の場合、Private Endpoint を再作成して IP が変わると**レコードの更新を忘れる**リスクがあります。必ず Private DNS Zone Group を使いましょう。
- Bicep / Terraform で IaC 管理している場合は、必ず `privateDnsZoneGroups`（Bicep）または `azurerm_private_dns_zone_group`（Terraform）をテンプレートに含めてください。

## トラブルシューティングに使えるコマンド集

以下は、Private Link + DNS のトラブルシューティング時に役立つコマンドをまとめたものです。

### DNS の名前解決確認

```powershell
# Windows - Resolve-DnsName（OS のリゾルバーを使用。推奨）
Resolve-DnsName myaccount.blob.core.windows.net

# Windows - nslookup（直接 DNS サーバーに問い合わせ）
nslookup myaccount.blob.core.windows.net

# Linux - dig（詳細な DNS 応答を確認）
dig myaccount.blob.core.windows.net

# Linux - host
host myaccount.blob.core.windows.net
```

### CNAME チェーンの確認

Private Endpoint が正しく構成されている場合、`privatelink` を含む CNAME が返ります。

```powershell
Resolve-DnsName myaccount.blob.core.windows.net -Type CNAME
# → myaccount.blob.core.windows.net → myaccount.privatelink.blob.core.windows.net
```

`privatelink` の CNAME が返らない場合、Private Endpoint の作成自体に問題がある可能性があります。

### ネットワーク疎通確認

```powershell
# TCP 接続テスト
Test-NetConnection -ComputerName myaccount.blob.core.windows.net -Port 443

# Linux
curl -v https://myaccount.blob.core.windows.net
nc -zv myaccount.blob.core.windows.net 443
```

### DNS キャッシュの管理

```powershell
# Windows - キャッシュ確認
Get-DnsClientCache | Where-Object { $_.Entry -like "*privatelink*" }

# Windows - キャッシュクリア
Clear-DnsClientCache

# Linux - systemd-resolved のキャッシュクリア
sudo systemd-resolve --flush-caches
```

### Private DNS Zone の状態確認 (Azure CLI)

```bash
# Private DNS Zone のレコード一覧
az network private-dns record-set list \
  --resource-group rg-network \
  --zone-name privatelink.blob.core.windows.net \
  --output table

# 仮想ネットワーク リンクの一覧
az network private-dns link vnet list \
  --resource-group rg-network \
  --zone-name privatelink.blob.core.windows.net \
  --output table

# Private Endpoint の DNS 構成確認
az network private-endpoint dns-zone-group list \
  --resource-group rg-app \
  --endpoint-name pe-storage01 \
  --output table
```

## まとめ

Private Link + Private DNS Zone のトラブルの多くは、**「DNS の名前解決が Private IP を返さない」** という点に集約されます。切り分けの基本は以下のフローです。

1. **`Resolve-DnsName` で FQDN を引いて、Private IP が返るか？**
  - 返らない場合は、DNS 構成の問題を疑います（パターン 1〜5, 7）。
  - 返る場合は、ネットワークまたはアプリ側の問題を疑います（パターン 6）。
2. **CNAME チェーンに `privatelink` が含まれるか？**
  - 含まれない場合は、Private Endpoint の構成自体を確認します。
  - 含まれる場合は、Private DNS Zone 側の問題を確認します。
3. **どこから名前解決しているか？**
  - VNet 内なら、Private DNS Zone のリンクとレコードを確認します。
  - オンプレミスや他 VNet なら、DNS フォワーダーの構成を確認します。

この順番で見ていくと、どこで名前解決が崩れているのかをかなり効率よく切り分けられます。

## 参考リンク

- [Azure プライベート エンドポイントの DNS 構成](https://learn.microsoft.com/ja-jp/azure/private-link/private-endpoint-dns)
- [Azure Private Link とは](https://learn.microsoft.com/ja-jp/azure/private-link/private-link-overview)
- [Azure DNS Private Resolver とは](https://learn.microsoft.com/ja-jp/azure/dns/dns-private-resolver-overview)
- [プライベート エンドポイントの DNS の問題のトラブルシューティング](https://learn.microsoft.com/ja-jp/azure/private-link/troubleshoot-private-endpoint-connectivity)
- [仮想ネットワーク リンクを使用した Azure プライベート DNS ゾーン](https://learn.microsoft.com/ja-jp/azure/dns/private-dns-virtual-network-links)
- [Hub-spoke ネットワーク トポロジにおけるプライベート エンドポイントの DNS 解決](https://learn.microsoft.com/ja-jp/azure/architecture/networking/guide/private-link-hub-spoke-network)
