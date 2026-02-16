---
title:  "GitHub Copilot・Azure Copilot・Azure MCP の違いを整理する"
date:   2027-02-17 14:34:25
categories: AI
author_profile: false
tags:
  - GitHub Copilot
  - Azure Copilot
  - Azure MCP
---

## こんなことを書いています

* GitHub Copilot、Azure Copilot（旧 Microsoft Copilot for Azure）、Azure MCP はそれぞれ役割が異なる
* 「どこで使うか」「何ができるか」を軸に整理
* 併用することで Azure 運用の生産性が大きく向上する

<!--more-->

## 3 つの "Copilot 系" サービス、何が違うの？

最近 AI 系のサービスが増えてきて、「GitHub Copilot」「Azure の Copilot」「Azure MCP」と似たような名前を耳にする機会が増えました。
それぞれ名前は似ていますが、使う場所も目的も異なります。本記事ではこの 3 つの違いを整理します。

## 比較表

| 項目 | GitHub Copilot | Azure Copilot | Azure MCP |
|---|---|---|---|
| **利用場所** | VS Code / JetBrains 等のエディタ | Azure Portal / Azure Mobile App | VS Code（GitHub Copilot の拡張） |
| **主な対象** | ソースコード全般 | Azure リソースの管理・運用 | Azure リソースの操作を AI エージェントから実行 |
| **できること** | コード補完、チャット、コードレビュー、テスト生成 | リソース情報の確認、トラブルシュート、コスト分析、設定変更 | リソースのデプロイ、構成変更、ログ取得などをエディタ内から直接実行 |
| **入力方法** | コード記述 / チャット | 自然言語チャット | GitHub Copilot チャットから自然言語で指示 |
| **課金** | GitHub Copilot ライセンス | Azure サブスクリプション（追加費用なし） | GitHub Copilot ライセンス + Azure サブスクリプション |

## GitHub Copilot

GitHub Copilot は、エディタ（VS Code、JetBrains 等）に統合された AI コーディングアシスタントです。

### 主な機能

* **コード補完** : コードを書いている最中にリアルタイムで候補を提示
* **Copilot Chat** : チャット形式でコードの説明、リファクタリング、バグ修正の提案を受けられる
* **エージェントモード** : 複数ファイルにまたがる変更やターミナル操作を自律的に実行
* **コードレビュー** : Pull Request のレビューを AI が支援

### ポイント

GitHub Copilot はあくまで **コーディングの生産性向上** が目的です。Azure に限らず、あらゆる言語・フレームワークで利用できます。
Bicep や ARM テンプレートを書く際にも補完が効くので、Azure インフラのコード化（IaC）にも役立ちます。

[GitHub Copilot 公式](https://github.com/features/copilot)

## Azure Copilot（旧 Microsoft Copilot for Azure）

Azure Copilot は、**Azure Portal 上で利用できる AI アシスタント** です。
Azure Portal の右上にある Copilot アイコンからチャットを開き、自然言語で Azure リソースに関する質問や操作を行えます。

> **名称の変遷**: Microsoft Copilot for Azure → Copilot in Azure → Azure Copilot と名前が変わっています。公式ドキュメントやトレーニング教材に旧名称が残っている場合がありますが、同じサービスです。

### 主な機能

* **リソース情報の確認** : 「このサブスクリプションの VM 一覧を見せて」のような質問に回答
* **トラブルシューティング** : 「この VM が起動しない原因は？」といった障害調査を支援
* **コスト分析** : 「先月のコストが高かったリソースは？」などコスト最適化のアドバイス
* **設定ガイド** : 「NSG でポート 443 を開放する方法は？」のような設定手順の案内
* **Azure Resource Graph クエリ生成** : 自然言語からクエリを生成

### ポイント

Azure Copilot は **Azure の運用・管理の効率化** が目的です。Azure Portal の中で完結するため、インフラエンジニアや運用担当者が日常的に利用するイメージです。
あくまでポータル上の操作支援なので、コードを書く場面では GitHub Copilot の出番になります。

[Azure Copilot 公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/copilot/overview)

## Azure MCP（Model Context Protocol）

Azure MCP は、**GitHub Copilot などの AI エージェントに Azure 操作の能力を追加する仕組み** です。
MCP（Model Context Protocol）というオープンな規格を利用して、エディタ内の AI エージェントから Azure リソースを直接操作できるようになります。

### 主な機能

* **リソースのデプロイ・管理** : 「Storage Account を East US に作成して」のような指示でリソースを作成
* **ログ・メトリクス取得** : Application Insights や Monitor のデータをエディタ内で確認
* **Azure CLI / Bicep の生成と実行** : 自然言語の指示から Azure CLI コマンドや Bicep テンプレートを生成し、そのまま実行
* **各種 Azure サービスとの連携** : App Service、Functions、Cosmos DB、Storage、Key Vault など多数のサービスに対応

### ポイント

Azure MCP は **エディタ（VS Code）から離れずに Azure リソースを操作** できる点が最大の特徴です。
GitHub Copilot のエージェントモードと組み合わせることで、「コードを書く → Azure にデプロイする → ログを確認する」という一連のフローをエディタ内で完結できます。

利用するには VS Code の Azure 拡張機能をインストールし、MCP サーバーの設定を行います。

[Azure MCP Server（GitHub）](https://github.com/Azure/azure-mcp)

## 紛らわしい「Microsoft Copilot」との違い

ここまで紹介した 3 つとは別に、**Microsoft Copilot**（copilot.microsoft.com）という汎用の AI チャットサービスもあります。
Bing Chat の後継にあたるもので、ブラウザや Windows、モバイルアプリから利用でき、検索・要約・画像生成など幅広い用途に対応します。

Azure に特化した機能は持っていないため、Azure リソースの管理には Azure Copilot を、コーディングには GitHub Copilot を使う必要があります。

| サービス | ざっくり一言 |
|---|---|
| **Microsoft Copilot** | 汎用 AI チャット（旧 Bing Chat）。Azure 特化ではない |
| **Microsoft 365 Copilot** | Word / Excel / Teams 等の Office アプリに統合された AI。業務文書向け |
| **Azure Copilot** | Azure Portal 上の AI アシスタント。Azure リソース管理に特化 |
| **GitHub Copilot** | エディタ上の AI コーディングアシスタント |

「Copilot」と名がつくサービスは多いですが、**対象領域（汎用 / Office / Azure / コーディング）** で整理するとわかりやすくなります。

## GitHub Copilot Extensions と Azure MCP の違い

GitHub Copilot の機能を拡張する仕組みとして、**GitHub Copilot Extensions** と **MCP（Model Context Protocol）** の 2 つがあります。Azure MCP は後者の仕組みを利用しています。

| 項目 | GitHub Copilot Extensions | Azure MCP（MCP サーバー） |
|---|---|---|
| **規格** | GitHub 独自の拡張フレームワーク | Anthropic 発のオープン規格（MCP） |
| **利用場所** | GitHub.com / VS Code / JetBrains 等 | VS Code（エージェントモード） |
| **導入方法** | GitHub Marketplace からインストール | MCP サーバーの設定（`mcp.json`）を追加 |
| **提供元の例** | Docker、Sentry、Azure（@azure）等 | Azure MCP Server、各種サードパーティ |
| **特徴** | GitHub エコシステムとの統合が強い | エディタ内でツール呼び出しとして動作し、エージェントが自律的に活用 |

どちらも「GitHub Copilot に外部ツールの能力を追加する」という点では似ていますが、仕組みが異なります。

- **GitHub Copilot Extensions** : GitHub のチャットで `@azure` のようにメンションして呼び出す形式。GitHub Marketplace で管理される
- **Azure MCP** : VS Code のエージェントモードで利用。MCP というオープンプロトコルに基づき、AI エージェントが必要に応じて自動的にツールを呼び出す

Azure 操作に関しては、VS Code のエージェントモードで本格的に使いたいなら **Azure MCP**、GitHub.com 上のチャットから手軽に Azure の情報を聞きたいなら **GitHub Copilot Extensions の @azure** という使い分けになります。

## Microsoft Learn MCP も別物

Azure MCP と名前が似ていて混同しやすいものに **Microsoft Learn MCP** があります。これは Azure リソースを操作するものではなく、**Microsoft Learn（旧 Microsoft Docs）のドキュメントを AI から直接検索・参照できる MCP サーバー** です。

| 項目 | Azure MCP | Microsoft Learn MCP |
|---|---|---|
| **目的** | Azure リソースの操作（デプロイ、ログ取得等） | Microsoft 公式ドキュメントの検索・参照 |
| **対象データ** | Azure サブスクリプション内のリソース | learn.microsoft.com 上のドキュメント |
| **サーバー種別** | ローカル（STDIO） | リモート（HTTP） |
| **エンドポイント** | ローカルで `azmcp` を起動 | `https://learn.microsoft.com/api/mcp` |
| **ユースケース** | 「Storage Account を作成して」「ログを確認して」 | 「Bicep の書き方を調べて」「App Service のベストプラクティスは？」 |

Microsoft Learn MCP を併用すると、**公式ドキュメントを確認しながらコードを書き、Azure MCP でそのままデプロイする** といった流れがエディタ内で完結します。

[Microsoft Learn MCP（GitHub）](https://github.com/microsoftdocs/mcp)

> なお、Microsoft が公式に提供している MCP サーバーは他にも Azure DevOps MCP、AKS MCP、Microsoft Fabric MCP、Microsoft Sentinel MCP 等があります。一覧は [microsoft/mcp リポジトリ](https://github.com/microsoft/mcp) で確認できます。

## 使い分けのイメージ

それぞれのツールは競合するものではなく、**場面に応じて併用** するのがベストです。

| シーン | 使うツール |
|---|---|
| Bicep テンプレートを書きたい | GitHub Copilot |
| Azure Portal でリソースの状態を確認したい | Azure Copilot |
| VS Code からリソースをデプロイしたい | Azure MCP + GitHub Copilot |
| Pull Request のコードレビューをしたい | GitHub Copilot |
| Azure のコストを分析したい | Azure Copilot |
| アプリのログを確認しながらコードを修正したい | Azure MCP + GitHub Copilot |

### フローで考える

```
開発フェーズ
  └─ GitHub Copilot でコードを書く
       └─ Azure MCP でリソースをデプロイ
            └─ Azure MCP でログ・メトリクスを確認

運用フェーズ
  └─ Azure Copilot でリソース状態を確認
       └─ Azure Copilot でトラブルシュート
            └─ 必要に応じて GitHub Copilot でコード修正
```

## まとめ

* **GitHub Copilot** : エディタ上の AI コーディングアシスタント。コードを書くならこれ
* **Azure Copilot** : Azure Portal 上の AI アシスタント。Azure の運用・管理に特化
* **Azure MCP** : GitHub Copilot に Azure 操作能力を追加する拡張。エディタから Azure を直接操作

名前は似ていますが、それぞれ活躍する場所が違います。
「コードを書くとき」「ポータルで管理するとき」「エディタから Azure を操作するとき」で使い分けると、Azure 運用の生産性が大きく向上するはずです。
