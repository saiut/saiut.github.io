---
title:  "Application Gateway の TCP プロトコルサポートを試してみる（プレビュー）"
date:   2024-05-22 14:34:25
author_profile: false
draft: true
categories: Network
---

## こんなことを書いています

* Application Gateway では AFD や TR で可能な「トラフィックの重みづけでの負荷分散」ができない
* 作りこみで「疑似」重みづけをやってみる

## NSG 使う

トラフィックの負荷分散で重みづけができるサービスは、Azure Front Door や Traffic Manager になります。
この 2 つのサービスは