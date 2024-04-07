---
title:  "Azure Update Manager のアップデート方法の違い"
date:   2024-04-12 14:34:25
categories: Operation
author_profile: false
tags:
  - "Update Manager"
---

## 結論

* ANF を SMB 接続する際にいきなりエラーになります
* Microsoft Entra Domain Services の設定を行うユーザーにも注意

## Architecture

アーキテクチャはこのような構成図になります。
![Architecture](/assets/article_images/2024-02-15-anf-entrads/architecture.png)