---
title: "HTTP-01の確認先はアプリのトップページではない"
emoji: "🔎"
type: "tech"
topics: ["security", "tls"]
published: true
---

ブラウザーでトップページが開くのに、ACMEのHTTP-01が失敗することがあります。利用者が見る画面と、CAが確認する専用パスは同じ条件とは限りません。URL、入口、応答する担当を分けて考えます。

![応答を準備: クライアントが検証用の値をHTTPの専用パスへ用意。 CAが取得: CAが対象名のポート80から指定パスを確認する。 条件を照合: トップページではなく、期待するチャレンジ応答と比べる。](/images/b074-http-01.png)

説明用の概念図。実測結果ではありません。

## CAが確認する専用パスを知る

HTTP-01では、ドメインのHTTPサーバーに、チャレンジ用の値を指定パスで応答させます。トップページのHTMLが返ることでは確認になりません。

Webアプリが存在しないパスをすべてトップページへ書き換える構成では、見た目は正常でも、要求されたチャレンジの応答と一致しない可能性があります。

## ポート80からの入口を確認する

Let's EncryptのHTTP-01はポート80を使います。リダイレクトについても許容する方式や行き先に規則があります。443のアプリが正常なだけでは、検証の入口が通るとは限りません。

ファイアウォール、CDN、ロードバランサー、Webサーバーのどこで応答が作られるかを整理します。認証画面やWAFが専用パスを遮っていないかも候補になります。

## 複数の宛先がある構成に注意する

名前が複数のサーバーへ向く場合、検証要求がどのサーバーへ届いても正しい応答が必要になります。自分が見た一台だけ成功した結果を、全経路へ広げないでください。

外部からの到達性を確認する担当と、チャレンジを配置する担当を分け、対象の名前・パス・時刻をそろえて調べます。

## 公開パスの確認と秘密情報の扱い

検証用の応答はCAが取得できる場所へ置きますが、ACMEアカウント鍵やDNS APIの秘密値をそこへ置くわけではありません。チャレンジの値と認証情報を混ぜないようにします。

本番で試行を連打する前に、検証用環境とクライアントの公式手順を使って経路を確認します。最終的に残すのは、トップページのスクリーンショットではなく、専用パスが正しく返ったという限定した記録です。

## 一次資料

- [Let's Encrypt: Challenge types](https://letsencrypt.org/docs/challenge-types/)
- [RFC 8555: ACME](https://www.rfc-editor.org/rfc/rfc8555.html)

## もう少し詳しく知りたい方へ

本文で扱った仕組みや運用を、図解や関連する確認項目まで掘り下げる資料です。

- [ACME通信を実測で読む：Accountから証明書取得まで](https://pki-channel.com/ja/articles/acme-protocol-walkthrough?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b074-1)
- [ACME導入前チェック：DNS・通信・ファイル・期待結果](https://pki-channel.com/ja/articles/acme-deployment-preflight?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b074-2)

---

資料確認・執筆・図解制作にAIを使用しています。例や確認表は説明用で、実機試験の結果ではありません。

