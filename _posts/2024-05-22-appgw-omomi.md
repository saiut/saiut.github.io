---
title:  "Application Gateway で「疑似」重みづけをしよう"
date:   2024-05-22 14:34:25
author_profile: false
draft: true
categories: Network
---

## こんなことを書いています

* Application Gateway では AFD や TR で可能な「トラフィックの重みづけでの負荷分散」ができない
* 作りこみで「疑似」重みづけをやってみる

## AFD と TM でもいいんだけど

トラフィックの負荷分散で重みづけができるサービスは、Azure Front Door や Traffic Manager になります。
この 2 つのサービスはグローバルサービスになるので、特定のリージョンに依存せず、パブリックに公開されるサービスになります。

こうなると、もしパブリックに公開させたくない要件のあるシステムの場合は利用できないため、 AFD や TM は残念ながら利用できません。

L7 トラフィックの負荷分散を行うことが出来る Azure のサービスは Application Gateway になりますが、AppGW は重みづけをつけたトラフィックのルーティングができません。

今回は、Application Gateway を利用して疑似的にトラフィックの重みづけをする方法を考えてみます。

## 思いついた方法 2 つ

パッと思いつく方法はこんな形でしょうか。

* VM 2 台のうち、VM2 をシャットダウンしておいて、VM1 側に障害があった際に立ち上げる

![architecture](/assets/article_images/2024-05-22-appgw-omomi/VMshutdown.png)

* AppGW で VM2 にもルーティングされるように設定しておき、普段は VM2 にルーティングされないように NSG をつけておいて、VM1 障害時に VM2 の NSG を外す

![architecture2](/assets/article_images/2024-05-22-appgw-omomi/nsg.png)

両方とも不可能ではなさそうです。 VM1 側で障害を検知したら、VM2 を立ち上げる or VM2 NIC に付随の NSG を外すだけで OK ですからね。
ただし、VM2 を立ち上げるのと NSG を外す動作だと、NSG を外すほうが時間的にはかからないので、今回は後者の方法をとってみたいと思います。

他に Application Gateway のバックエンドプールに入れるなどの方法もありますが、Application Gateway 自体を操作する必要がありますので、それよりも簡易な方法の NSG のほうがいいかなと思います。（ただしもちろんデメリットもありますのでそれは後述）

## NSG を外す

では、VM1 側に障害があった際に、VM2 の NSG を外して VM2 側にルーティングする方法を考えていきます。
今回の場合は 2 台なので、アラート発報の契機としては、AppGW の正常なホストの数でよさそうです。

AppGW では、メトリックとして正常なホスト数（Healthy Host Count）というのを取得していて、その正常なホストがなくなったときに VM2 についている NSG を外すようにします。

ちなみに正常「ではない」ホスト数に関しては Unhealthy Host Count というメトリックになります。

![metric](/assets/article_images/2024-05-22-appgw-omomi/metric.png)

上記の画像だと、2 台の正常なホストがいて、とあるタイミングで 1 台に異常があって正常なホストが 1 に減っています。

今回のやりたいことだと、2台構成の Active-Standby だと、正常なホスト数が1台で、それ以下になった場合はアラートが発生して VM2 が動作する流れになります。

アラート発生時に NSG を外すには、アラートのアクションとして Automation を利用して、アラート発生時に Automation が作動して NSG を外す流れです。

サンプルとしてはこのような形でしょうか

``` powershell
$AzureContext = Connect-AzAccount -Identity

## Place the network interface configuration into a variable. ##
$nic = Get-AzNetworkInterface -Name <nicname> -ResourceGroupName <RGname>

## Remove the NSG to the NIC configuration. ##
$nic.NetworkSecurityGroup = $null

## Save the configuration to the network interface. ##
$nic | Set-AzNetworkInterface
```

NSG を外す NIC を指定し、その値を NULL とする = NSG を外すという方法です。
なので、あくまで NIC につけている NSG は、AppGW からのルーティングされない設定だけ追加（AppGW からの HTTP,HTTPS を受け付けない）しておき、その他セキュリティとして必要な NSG はサブネットの NSG で設定するようにしておきます。

## デメリット