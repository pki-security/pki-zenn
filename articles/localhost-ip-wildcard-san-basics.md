---
title: "localhost・IPアドレス・ワイルドカード 証明書の名前で迷わない設計表"
emoji: "🔐"
type: "tech"
topics: ["pki", "tls", "openssl", "初心者"]
published: true
---

開発環境をHTTPS化するとき、最初に決めたいのは証明書のファイル名ではなく、ブラウザやAPIクライアントが実際に開くURLです。`localhost`、IPアドレス、独自のDNS名は、証明書の名前照合で同じものとして扱えません。

## 接続先の名前と接続先の場所

DNSは名前から接続先のIPを調べる仕組みです。一方、証明書の名前照合は、その接続相手がURLの名前に対応するかを調べます。IPへ到達できても、名前照合が成功するとは限りません。[RFC 9525 — Service Identity in TLS](https://www.rfc-editor.org/rfc/rfc9525.html)

|URLの例|証明書に必要な識別子の例|注意点|
|---|---|---|
|`https://localhost/`|DNS名としてlocalhost|127.0.0.1のIP欄とは別|
|`https://127.0.0.1/`|IPアドレスとして127.0.0.1|DNS名の文字列で代用しない|
|`https://app.demo.test/`|DNS名としてapp.demo.test|名前解決も別途必要|
|`https://a.b.demo.test/`|その名前を覆うDNS-ID|`*.demo.test` だけでは不足|

これはSANという証明書の欄で表す情報の例です。名前が合うことに加え、クライアントが発行者を信頼すること、有効期間などの条件も必要です。

名前の候補が増えて迷ったら、PKIちゃんねるの[ワイルドカードとSANの対応表](https://pki-channel.com/ja/articles/what-is-wildcard-cert)で、対象名と対象外の名前を並べて確認できます。複数サービスで鍵を共有する場合の範囲も整理できます。

## ワイルドカードの範囲を実験する

手元の学習用教材で作成した `server.pem` は `demo.test` と `*.demo.test` をSANに持っています。2026年10月30日にローカルの学習用CAと中間CAを用い、OpenSSL 3.5.4で照合名を変えて検証したところ、次の結果になりました。実行例は `openssl verify -purpose sslserver -CAfile root.pem -untrusted intermediate.pem -verify_hostname api.demo.test server.pem` です。自分の証明書で試す場合は、各ファイル名と信頼設定を実際の環境に合わせてください。

|照合名|結果|理由|
|---|---|---|
|demo.test|成功|完全一致のSANを別に含む|
|api.demo.test|成功|ワイルドカードの一階層分に一致|
|a.b.demo.test|名前不一致|二階層分は覆わない|

`demo.test` の成功を「ワイルドカードが親ドメインも覆う」と解釈しないでください。今回は完全一致のSANも入っているためです。この違いは、証明書の中身を読むことで確認できます。

```sh
openssl x509 -in server.pem -noout -ext subjectAltName
```

決めた名前を発行依頼に落とす段階では、[CSRから発行後までの確認表](https://pki-channel.com/ja/articles/csr-best-practices)を使えます。要求したSANと、届いた証明書のSANを照合する担当まで決められます。

## チームの開発環境で先に決めること

自分の端末だけで動けばよい場合と、複数人のPC・スマホから同じ環境へ接続する場合では、必要なURLが違います。スマホの `localhost` はスマホ自身なので、開発PCのサーバーにはなりません。

各利用者のURL、名前解決、証明書のSAN、信頼の配布方法を一行ずつ書きます。学習用CAの鍵をメンバー全員へコピーすると、どの名前でも証明書を発行できる権限まで配ることになり得ます。CA鍵の保管者と証明書の配布者を分けて考えます。

```text
利用端末：テスト用スマホ
入力するURL：https://app.example.test
名前を解決する方法：________
証明書のSAN：________
その端末が信頼する根拠：________
使い終わった信頼設定を戻す担当：________
```

ホスト名検証を無効化する設定を共有するより、チームでURLを統一した方が、Cookieやリダイレクトの確認条件もそろえやすくなります。証明書の名前の設計は、アプリが使う名前の設計でもあります。

## 一次資料

- [RFC 9525 — Service Identity in TLS](https://www.rfc-editor.org/rfc/rfc9525.html)
- [OpenSSL 3.5 — openssl-x509](https://docs.openssl.org/3.5/man1/openssl-x509/)
- [RFC 5280 — Certificates and CRLs](https://www.rfc-editor.org/rfc/rfc5280.html)
