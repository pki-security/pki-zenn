---
title: "ACME Orderのreadyは発行完了ではない：状態名を順番に読む"
emoji: "🔎"
type: "tech"
topics: ["security", "tls"]
published: true
---

ACMEのログにreadyやvalidと出ていても、何のオブジェクトの状態なのかで意味が変わります。ここではOrderに絞り、どこまで進んだかを説明できるようにします。サーバーへの配備は、Orderが完了した後の別の工程です。

![pending → ready: 必要な認可がそろうと、注文の確定へ進める。 finalize → processing: CSRを送り、CAが発行処理を行う。失敗経路もある。 valid → その後: 発行完了後に取得・配備・接続確認を行う。](/images/b073-acme-order.png)

説明用の概念図。実測結果ではありません。

## Orderは発行の注文

Orderは、希望する識別名について証明書を取得する一つの注文を表します。アカウント、Authorization、Challengeとは役割が違います。

ログを読むときは、状態名の前にオブジェクトの種類とURLを確認します。Challengeのvalidを見て、Orderまでvalidになったと早合点しないためです。

## pendingからreadyへ

pendingでは、必要な認可など、発行の準備がそろうのを待ちます。readyは、注文を確定する処理へ進める状態です。証明書ファイルを取得できたという意味ではありません。

どの識別名に未完了の認可があるかを調べ、同じ注文をむやみに作り直す前に、現在の状態を確認します。

## finalizeの後はprocessingとvalid

クライアントがCSRを伴うfinalize要求を送り、CAが発行処理を行うとprocessingを経てvalidへ進みます。発行処理に失敗するなどしてinvalidとなる経路もあります。

これは概念上の状態遷移です。実際のポーリングでは短い中間状態を観測しない場合もあります。ログにすべての状態が並ばないことだけで異常とは言えません。

## validの後に運用の完了がある

Orderがvalidでも、証明書の取得、保存、配備、サービスの再読み込み、接続先での確認は残ります。

報告を「注文状態／取得済みか／配備済みか／接続確認済みか」の四欄にすると、CAの発行完了と利用者が新しい証明書を見る時点を分けられます。これは更新障害の担当分けにも使える小さな状態表です。

## 一次資料

- [RFC 8555: ACME](https://www.rfc-editor.org/rfc/rfc8555.html)

## もう少し詳しく知りたい方へ

本文で扱った仕組みや運用を、図解や関連する確認項目まで掘り下げる資料です。

- [ACME通信を実測で読む：Accountから証明書取得まで](https://pki-channel.com/ja/articles/acme-protocol-walkthrough?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b073-1)
- [ACME更新運用：発行・配置・reload・配信確認](https://pki-channel.com/ja/articles/acme-renewal-deployment-design?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b073-2)

---

資料確認・執筆・図解制作にAIを使用しています。例や確認表は説明用で、実機試験の結果ではありません。
