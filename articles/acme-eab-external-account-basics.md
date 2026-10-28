---
title: "ACMEのEABとは？CAの契約アカウントとACMEアカウントを結ぶ入口"
emoji: "🔎"
type: "tech"
topics: ["security", "tls"]
published: true
---

ACMEを設定したらEABのKey IDとHMAC keyを求められた。これはサーバー証明書の秘密鍵でしょうか。EABはExternal Account Binding、外部アカウントとACMEアカウントを結び付けるための仕組みです。

![外部アカウント: CAが契約などのアカウントに対応するEAB情報を渡す。 ACME登録: クライアントが新しいACMEアカウントとの結び付きを示す。 別の確認: ドメイン検証と証明書の鍵管理は別に行う。](/images/b076-acme-eab.png)

説明用の概念図。実測結果ではありません。

## 二つのアカウントを結び付ける

CA側にある顧客・契約などの外部アカウントと、新しく登録するACMEアカウントを関連付けたい場合があります。EABはその関連付けに使います。

ドメインの管理権限を示すHTTP-01やDNS-01とは役割が違います。EABが通ったから対象ドメインの検証がすべて済む、という意味ではありません。

## Key IDとHMAC keyを同じものと思わない

Key IDは結び付けに使う鍵を識別する値、HMAC keyは要求の認証に使う秘密値です。表示名が似ていても、両者の扱いは異なります。

秘密値をチャットやソース管理へ貼り付けず、製品が指定するSecretや秘密管理の経路へ渡します。説明用の設定例と本物の値を明確に分けます。

## 登録先と環境をそろえる

CAのテスト環境と本番環境で、ACME DirectoryのURLやEAB情報が異なる場合があります。取り違えると、値を正しく入力したつもりでも登録できません。

エラーを調べるときは、秘密値そのものではなく、環境名、登録先、Key IDの管理情報、エラー種別を確認します。

## 設定が成功した後も残る管理

EAB情報、ACMEアカウント鍵、証明書の秘密鍵、DNS APIトークンをそれぞれ別の役割として管理します。失効・再発行・回収の条件は、利用CAの現在の案内を確認してください。

初回担当者だけが知っている値にしないことが大切です。次の担当者が「どの契約に、どのACMEアカウントが結び付いているか」を秘密値を見ずに説明できる台帳を残します。

## 一次資料

- [RFC 8555: ACME](https://www.rfc-editor.org/rfc/rfc8555.html)
- [cert-manager: ACME configuration](https://cert-manager.io/docs/configuration/acme/)

## もう少し詳しく知りたい方へ

本文で扱った仕組みや運用を、図解や関連する確認項目まで掘り下げる資料です。

- [ACME EAB：契約アカウントとの連携・設定・障害対応](https://pki-channel.com/ja/articles/acme-eab?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b076-1)
- [ACMEとは：3分で理解し、仕組みまで学ぶ](https://pki-channel.com/ja/articles/acme-protocol-internals?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b076-2)

---

資料確認・執筆・図解制作にAIを使用しています。例や確認表は説明用で、実機試験の結果ではありません。
