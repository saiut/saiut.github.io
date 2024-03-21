---
title:  "Virtual WAN で仮想ハブが相互接続されているときの VNet の動作"
date:   2024-03-10 14:34:25
author_profile: false
categories: Network
tags:
  - VWAN
  - VNet
header:
  overlay_image: /assets/article_images/2023-09-30-vwan-vhub-vnet/vwan-architecture.png
sidebar:
  toc: true
---

## TL;DR

- 同じ Virtual WAN 内の仮想ハブは暗黙で Peering が張られて、お互いの仮想ハブに接続されているネットワークは相互に通信が可能
- 仮想ハブについては VNet を接続できるが、違う仮想ハブであれば同じ IP Range の VNet を接続できる
- 接続できてしまうが、対象の VNet 同士は通信できないので注意
- 同一仮想ハブ内であれば IP Range が被っていると接続できないとエラーが出るのに。。。

----

## Virtual WAN

Virtual WAN を活用することで、様々な場所から Auzre への接続を 1 つの Hub にまとめることができます。
以下のようなものを、1 つの Virtual WAN にまとめてしまい、相互に接続することが出来るサービスです。

- S2S VPN
- P2S VPN
- ExpressRoute
- VNet

公式ドキュメントからの抜粋ですが、以下のような形になります。
![VWAN-Architecture](/assets/article_images/2023-09-30-vwan-vhub-vnet/vwan-architecture.png)

大規模に様々な接続形態がある場合に使用されることが多い Virtual WAN ですが、
その中心にいるものが仮想ハブと言われるものです。

## 複数の仮想ハブ

仮想ハブは同一リージョン内に複数作成することも可能で、仮想ハブ同士は特に設定を意識することなくピアリングされ推移的な接続がなされます。

ですので、オンプレミス - 仮想ハブA - 仮想ハブB - VNet C といった接続を設定なしで通信してくれるようになります。

![OnP-VirtualHub-Vnet](/assets/article_images/2023-09-30-vwan-vhub-vnet/onp-virtualhub-vnet.jpg)

VNet の追加は非常に簡単で、Virtual WAN の設定より「接続の追加」から VNet を追加するだけです。

![VWAN-add-VNet](/assets/article_images/2023-09-30-vwan-vhub-vnet/vwan-add-vnet.jpg)

VNet を追加すると、 VNet 側には自動で VNet Peering が追加されます。

![VNet-Peering](/assets/article_images/2023-09-30-vwan-vhub-vnet/vnet-peering.jpg)

もちろん 同じ仮想ハブ内で複数の VNet やオンプレミスのネットワークを接続する際、 VNet の IP Range が被っているとエラーがでます。

![Same-IP Range](/assets/article_images/2023-09-30-vwan-vhub-vnet/same_iprange.jpg)

----

## VWAN で仮想ハブが複数存在する場合の IP Range には注意

そりゃそうですが、VNet 同士の IP Range が重複している場合、お互いに通信できることはできません。
IP Range は管理されていることが多いので、重複が発生することはほぼないと思います。
しかし、例えば Virtual WAN はとある部署が管理していて、そこに接続するためのシステムが別部署で管理している場合に　IP Range が被ってしまうことが起こり得てしまうでしょう。

そんな時に何も考えずに 仮想ハブに VNet を繋いでいくと、こんな構成が取れてしまいます。
![VWAN-Same-Ip-Range](/assets/article_images/2023-09-30-vwan-vhub-vnet/vwan-same-ip-range.jpg)

この場合、仮想ハブ A と仮想ハブ B に同じ 10.1.0.0/16 の IP Range を持つ VNet C と VNet D がありますが、
2 つの VNet 間はもちろん通信できません。

テストしてみると、VNet C と VNet D 以外の VNet （例えば 192.168.0.0/16）以外は通信可能なので、
なんでもかんでもとりあえずえいやで繋いでしまう構成はよくないってことがわかりますね。

## Appendix

Azure VWAN のテストをしたい場合は以下のテンプレートを使うのがいいでしょう。

[Azure Virtual WAN (vWAN) Multi-Hub Deployment](https://learn.microsoft.com/ja-jp/samples/azure/azure-quickstart-templates/virtual-wan-with-all-gateways/)