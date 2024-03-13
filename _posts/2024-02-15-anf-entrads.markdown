---
layout: posts
toc: true
title:  "Microsoft Entra Domain Service に Azure NetApp Files を接続するときにハマったこと"
date:   2024-02-22 14:34:25
categories: ANF
output:
    toc: true
---

# TL;DR

* Azure NetApp Files を Mirosoft Entra Domain Services に参加させるときにハマったことをまとめました
* Microsoft Entra Domain Services の設定を行うユーザーにも注意

# Architecture

アーキテクチャはこのような構成図になります。
![Architecture](/assets/article_images/2024-02-15-anf-entrads/architecture.png)

ANF のテナントと Entra Domain Services のテナントを分けていますが、深い意味はありません。
簡単に解説するとこのようなアーキテクチャです。

* オンプレミスを模倣した左側の PC は、オンプレ AD に所属
* クライアント PC は SMB プロトコルで Entra DS に参加している ANF をマウント
* オンプレミス AD は Entra ID Connect 経由で Entra ID に同期
* Entra ID に同期されたユーザーは Entra Domain Services に同期される

それぞれのサービスについては以下リンクを参照してください。

[Microsoft Entra Domain Services](https://learn.microsoft.com/ja-jp/entra/identity/domain-services/overview)

[Azure NetApp Files](https://learn.microsoft.com/ja-jp/azure/azure-netapp-files/azure-netapp-files-introduction)

# ハマったこと

Entra Domain Service は簡単にいうと「マネージドの Active Directory Domain Services」ですが、
マネージドなので ADDS とは異なる部分があります。
エンタープライズ管理者の特権がない、スキーマの拡張機能がないなど ADDS とは違う部分があるので、利用方法には注意が必要です。

比較についてはこちらをご確認ください。

[セルフマネージド Active Directory Domain Services、Microsoft Entra ID、マネージド Microsoft Entra Domain Services を比較する](https://learn.microsoft.com/ja-jp/entra/identity/domain-services/compare-identity-solutions#domain-services-and-self-managed-ad-ds)

ANF を SMB で利用したい場合には、ANF を ADDS に所属する必要があります。

![ADDS接続](/assets/article_images/2024-02-15-anf-entrads/connect-adds.png)

[Azure NetApp Files の Active Directory 接続の作成と管理](https://learn.microsoft.com/ja-jp/azure/azure-netapp-files/create-active-directory-connections)

その AD 接続を行った際に、Entra Domain Services ならではのハマったことがあったので、そちらを下記にいくつか記載します。

## Entra Domain Service の ADDS/DNS を見たい場合は AAD DC Administrators に所属しているユーザーでログイン

そもそも ANF をマウントする前に、Entra Domain Services の ADDS/DNS 機能を設定するには、Azure ポータルなどからは出来ず、別途 Entra Domain Services に接続できる VM などを構築する必要があります。

その際、対象の VM にログインするユーザーは AAD DC Administrators に所属している必要があります。

Entra ID でユーザーを作成してあげるか、ADDS で同期されてきた Entra ID に出来たユーザーに対して、 AAD DC Administrators に所属させてあげましょう。

![AAD DC Administrators](/assets/article_images/2024-02-15-anf-entrads/aaddcadministrators.png)

対象のユーザーで VM に対してログインし、 VM に ADDS のツールや DNS のツールをインストールして設定してあげましょう。
ちなみに DNS ツールを利用する際は、Entra ID に割り当てられた IP アドレスではなく、FQDN を利用しないといけないことに注意です。

## PTR レコードを Entra DS 環境の DNS に入れる

ADDS 接続を作成し、SMB ボリュームを作成する際、以下のエラーが発生しました。

```
>Failed to create the Active Directory machine account \"SMB-ANF-VOL. Reason: LDAP Error: Local error occurred Details: Error: Machine account creation procedure failed. [nnn] Loaded the preliminary configuration. [nnn] Successfully connected to ip 10.x.x.x, port 88 using TCP [nnn] Successfully connected to ip 10.x.x.x, port 389 using [nnn] Entry for host-address: 10.x.x.x not found in the current source: FILES. Ignoring and trying next available source [nnn] Source: DNS unavailable. Entry for host-address:10.x.x.x found in any of the available sources\n*[nnn] FAILURE: Unable to SASL bind to LDAP server using GSSAPI: local error [nnn] Additional info: SASL(-1): generic failure: GSSAPI Error: Unspecified GSS failure. Minor code may provide more information (Cannot determine realm for numeric host address) [nnn] Unable to connect to LDAP (Active Directory) service on contoso.com (Error: Local error) [nnn] Unable to make a connection (LDAP (Active Directory):contosa.com, result: 7643.
```

これは、ANF を AD 接続する際は、 PTR レコードが必要なために発生するエラーです。
あくまで検証環境なので、ADDS 環境はユーザーを作成したりするぐらいで、あまりいじっていなかったのが1つ原因かもしれません。
このエラーが発生した際は、 Entra Domain Services の DNS に AD サーバーの PTR レコードを登録してあげましょう。

## OU の指定が AADDC Computers

![ADDS接続](/assets/article_images/2024-02-15-anf-entrads/connect-adds.png)

AD 接続設定時、「組織単位のパス」を入れます。
これは ANF 自体をどの OU に所属させるかという設定になります。
特に値を設定しない場合は CN=Computers を利用しますが、 Entra DS の場合は *OU=AADDC Computers* である必要があります。

エラーとしてはこんな感じに出てしまいます。

```
{"code":"DeploymentFailed","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/DeployOperations for usage details.","details":[{"code":"InternalServerError","message":"Error when creating - Failed to create the Active Directory machine account \"SMBTESTAD-D9A2\". Reason: SecD Error: ou not found Details: Error: Machine account creation procedure failed\n [ 561] Loaded the preliminary configuration.\n [ 665] Successfully connected to ip 10.x.x.x, port 88 using TCP\n [ 1039] Successfully connected to ip 10.x.x.x, port 389 using TCP\n**[ 1147] FAILURE: Specifed OU 'OU=AADDC Com' does not exist in\n** contoso.com\n. "}]}
```

## AES 暗号化がされていない

Entra DS では、Kerberos RC4 暗号化が利用できますが、既定では無効化されています。
特に設定をしていない場合、ANF の AD 接続時にAES 暗号化にチェックを入れておく必要があります。
以下のようなエラーが発生します。

```
Failed to create the Active Directory machine account \"SMB-ANF-VOL\". Reason: Kerberos Error: KDC has no support for encryption type Details: Error: Machine account creation procedure failed [nnn]Loaded the preliminary configuration. [nnn]Successfully connected to ip 10.x.x.x, port 88 using TCP [nnn]FAILURE: Could not authenticate as 'contosa.com': KDC has no support for encryption type (KRB5KDC_ERR_ETYPE_NOSUPP)
```

# まとめ

今まで紹介したことが特に設定していない Entra DS に対して ANF の AD 接続を行う際にハマったことです。

上記に関しては、 ANF 側で AD 接続を行ったときには出ないエラーで、ボリューム作成時に SMB プロトコルで接続する設定を入れたときに発生するエラーなので、すぐには気付かず、
ボリューム作成時に「あれ？？」となるエラーになります。
なので、事前に上記のようなエラーとなりそうなものに対処してからボリューム接続に臨みましょう。

# Appendix

[Azure NetApp Files のボリュームに関するエラーをトラブルシューティングする](https://learn.microsoft.com/ja-jp/azure/azure-netapp-files/troubleshoot-volumes)