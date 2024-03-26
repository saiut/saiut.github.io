---
title:  "Logic Apps ワークフローのアクション分岐をまとめる"
date:   2024-05-03 14:34:25
categories: LogicApps
author_profile: false
tags:
  - PaaS
---

## TL;DR

* Logic Apps は既定でワークフローは順番で実行される
* 分岐をさせる方法をまとめました

## 分岐方法のいくつか

Logic Apps を利用してワークフローを実行させる場合、通常であれば順に沿って実行されます。

ただし、条件を付けて「特定の曜日の場合は実行させない」などといったことを実施したいことはあると思います。

その場合にどのような方法があるかをまとめました。

以下のような方法があります。

* Conditional
* Switch 
* Brunches
* Loop
* Scopes

### Conditional

まずは Conditional、条件です。
その名の通り、条件に合致すればxxxする、合致しなければ△△△する、といった条件分岐です。

### Switch 

### Brunches

### Loop

### Scopes

