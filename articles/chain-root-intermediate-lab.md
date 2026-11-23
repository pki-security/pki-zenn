---
title: "ルート証明書と中間証明書を手元で区別する 信頼の連鎖の小さな実験"
emoji: "🔐"
type: "tech"
topics: ["pki", "tls", "openssl", "初心者"]
published: true
---

証明書チェーンの説明では、ルート、中間、サーバーという三つの名前が出てきます。大切なのは並び順の暗記ではなく、利用側が何を最初から信頼し、何を受け取って検証するかです。

サーバーがルート証明書を送ってきても、それだけで信頼してよいことにはなりません。信頼点は、利用側があらかじめ選ぶものです。そうでなければ、誰でも自分のCAを送り付けて信頼させられてしまいます。[RFC 5280 — Certificates and CRLs](https://www.rfc-editor.org/rfc/rfc5280.html)

## 教材の三者

[root.pem](https://raw.githubusercontent.com/pki-security/pki-zenn/main/assets/beginner-chain-lab-20261030/root.pem)、[intermediate.pem](https://raw.githubusercontent.com/pki-security/pki-zenn/main/assets/beginner-chain-lab-20261030/intermediate.pem)、[server.pem](https://raw.githubusercontent.com/pki-security/pki-zenn/main/assets/beginner-chain-lab-20261030/server.pem)、[expired.pem](https://raw.githubusercontent.com/pki-security/pki-zenn/main/assets/beginner-chain-lab-20261030/expired.pem) を教材として使います。秘密鍵は含みません。OSの信頼ストアへ登録せず、OpenSSL 3.5系のコマンド内でだけ使います。

|ファイル|この実験での役割|誰が署名したか|
|---|---|---|
|root.pem|こちらが選んだ信頼点|自分自身|
|intermediate.pem|経路をつなぐCA証明書|ルートCA|
|server.pem|demo.test用のサーバー証明書|中間CA|

自己署名であることは、利用側が信頼する理由そのものではありません。この実験では `-CAfile root.pem` と指定することで、選択を明示しています。

手元の三つの証明書をPKIちゃんねるの[信頼経路と名前照合の図](https://pki-channel.com/ja/articles/tls-certificate-validation)へ対応させてみてください。ルートを信頼点として用意することと、中間証明書を経路候補として渡すことの違いを確認できます。

## 中間証明書を渡さない場合

```sh
openssl verify -attime 1793318400 -purpose sslserver -CAfile root.pem -no-CApath -no-CAstore -verify_hostname demo.test server.pem
```

OpenSSL 3.5.4で `unable to get local issuer certificate` を確認しました。ルートを知っていても、サーバー証明書を発行した中間CAが見つからず、経路をつなげられませんでした。

次は `server.pem` の前に `-untrusted intermediate.pem` を加えます。

```sh
openssl verify -attime 1793318400 -purpose sslserver -CAfile root.pem -no-CApath -no-CAstore -verify_hostname demo.test -untrusted intermediate.pem server.pem
```

今度は `server.pem: OK` でした。変えたのは中間証明書の候補を渡した点だけです。`-untrusted` は「このファイルを信頼点にしないが、経路を組み立てる材料に使う」という役割です。[OpenSSL 3.5 verify](https://docs.openssl.org/3.5/man1/openssl-verify/)

時刻指定は教材の再現用です。2026年10月30日00:00 UTCへ固定しているので、本番接続の成否や現在の有効性を示す結果としては使えません。

## なぜ一部の端末だけ失敗するのか

クライアントが持っている中間証明書や補完動作、信頼点、検証方針には差があります。ブラウザで一度開けたことだけを、サーバーのチェーン配信が完全な証拠にしないでください。

不具合調査では「動く端末」と「失敗する端末」の信頼ストアを無作為にそろえる前に、サーバーが何を送っているかを確認します。必要な中間証明書の配信漏れなら、サーバー側の設定を直すことが本来の対応になる場合があります。[curl — TLS certificate verification](https://curl.se/docs/sslcerts.html)

端末ごとの出発点をもう少し知りたい場合は、[ルートストアの仕組み](https://pki-channel.com/ja/articles/what-is-root-store)へ進めます。証明書を配る側の採用判断と、接続時の検証を別々に説明しています。

## 理解を確かめる問い

「中間証明書を信頼点へ直接追加すればよいか」と聞かれたら、まず意図を確認します。中間CAを新たな信頼点として扱う変更と、サーバーから経路候補を送る変更は、別の操作だからです。

この実験で確認できたのは、固定した時刻・名前・用途・信頼点に対して経路を検証した結果です。Webサイトの中身の安全性や、すべての端末での互換性までは判定していません。何を信頼して、何を検証したかをセットで説明できれば、チェーンの理解が設定の判断につながります。

## 一次資料

- [RFC 5280 — Certificates and CRLs](https://www.rfc-editor.org/rfc/rfc5280.html)
- [OpenSSL 3.5 verify](https://docs.openssl.org/3.5/man1/openssl-verify/)
- [curl — TLS certificate verification](https://curl.se/docs/sslcerts.html)
