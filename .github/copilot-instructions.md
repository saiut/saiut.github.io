# ブログ プロジェクト ガイドライン

## 概要

Azure インフラ領域の検証や Tips を紹介する日本語技術ブログ。Jekyll + Minimal Mistakes テーマで構築し、GitHub Pages でホスティング。

## 記事フロントマター規約

```yaml
---
title:  "記事タイトル"
date:   YYYY-MM-DD HH:MM:SS
categories: カテゴリ名
author_profile: false
tags:
  - タグ1
  - タグ2
header:
  teaser: /assets/article_images/YYYY-MM-DD-slug/image.jpg
---
```

### 必須フィールド
- `title`: 日本語、内容を端的に示す
- `date`: 投稿日時（タイムゾーン: Asia/Tokyo）
- `categories`: 1つのカテゴリ（例: ネットワーク, DR, アプリケーション, 監視）
- `tags`: Azure サービス名など具体的なタグを配列で指定

### オプション
- `author_profile`: 個別記事では `false`
- `header.teaser`: 記事固有の teaser 画像パス（なければデフォルトが適用される）

## ファイル命名規則

- **公開記事**: `_posts/YYYY-MM-DD-slug.md`
- **下書き**: `_drafts/YYYY-MM-DD-slug.md`
- **画像**: `assets/article_images/YYYY-MM-DD-slug/`

slug はタイトルを英語で簡潔にハイフン区切りで表現する。

## 記事構成テンプレート

1. **冒頭要約**: `## こんなことを書いています` で箇条書き
2. **`<!--more-->`**: excerpt 区切り（SEO 用）
3. **本文**: `##` 見出しで階層的に構成
4. **参考リンク**: Microsoft Learn 公式ドキュメントを `[タイトル](URL)` で参照
5. **画像**: `![alt](/assets/article_images/YYYY-MM-DD-slug/filename.jpg)` 形式

## 文体

- 「です・ます」調
- 技術的に正確で、実装時のハマりポイントを重視
- コード例（JSON, PowerShell, KQL 等）を積極的に含める
- 日本語として自然な表現を使う

## ビルド・プレビュー

```bash
bundle exec jekyll serve
```

サイト URL: https://tech.saiut.com
