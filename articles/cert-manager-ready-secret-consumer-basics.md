---
title: "cert-managerのCertificateがReadyでも終わらない Secretから利用先まで読む"
emoji: "🔐"
type: "tech"
topics: ["kubernetes", "certmanager", "tls"]
published: true
---

cert-managerのCertificateに `Ready=True` が付いたのに、ブラウザが古い証明書を表示することがあります。Certificateは証明書の希望条件を記述し、cert-managerは証明書と秘密鍵をSecretへ保存します。そのSecretをどのサービスが読むかは、さらに別の設定です。[cert-manager — Certificate resource](https://cert-manager.io/docs/usage/certificate/)

ここではKubernetesとcert-managerが導入済みで、利用可能なIssuerを管理者が用意している環境を想定します。以下のYAMLは構造を読むための例で、そのまま新しいクラスタへ適用する導入手順ではありません。

## 四つの名前を読む

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: shop-certificate
  namespace: web
spec:
  secretName: shop-tls
  dnsNames:
    - shop.example.com
  issuerRef:
    name: public-ca
    kind: ClusterIssuer
```

`metadata.name` はCertificateリソースの名前、`secretName` は保存先Secret、`dnsNames` は証明書に欲しいDNS名、`issuerRef` は発行元の参照です。同じ文字列にそろえる必要はありませんが、どれが何の名前かを混同すると調査が止まります。

Issuerは名前空間内、ClusterIssuerはクラスタ全体から参照するリソースです。この例をIssuerに変えるなら、参照先は同じ名前空間に必要です。名前が似ていても、kindが違えば別物です。[cert-manager — Issuer](https://cert-manager.io/docs/concepts/issuer/)

リソースの関係をもう一段追いたいときは、PKIちゃんねるの[Secretを利用する側まで含めたcert-manager解説](https://pki-channel.com/ja/articles/cert-manager-kubernetes#secret)へ進めます。Certificateの状態と、Ingressなどが読み込んだ結果を分けて確認できます。

## 秘密鍵を表示せず状態を見る

```sh
kubectl -n web get certificate shop-certificate
kubectl -n web describe certificate shop-certificate
kubectl -n web get secret shop-tls
kubectl -n web get ingress
```

最初は状態・イベント・参照先だけを見ます。Secretを `-o yaml` で丸ごと表示すると、秘密鍵を含むデータを端末ログへ残す可能性があります。base64で表示されることは暗号化されていることを意味しません。

## 確認する順番

|状態|調べる場所|早まらない対応|
|---|---|---|
|Certificateが未Ready|Certificateのイベント、Issuer、発行要求|Secretを手で何度も作り直さない|
|Secretはあるが利用されない|IngressやアプリのSecret参照と名前空間|CAの認証を無関係に繰り返さない|
|一部だけ古い証明書|接続先、コントローラー、再読み込み動作|クラスタ全体を理由なく再起動しない|

IngressのTLS設定で参照するSecret名と、Certificateの `secretName` が一致しているかを確認します。また、別のCDNやロードバランサーが利用者側のTLSを終端していれば、そこが出す証明書も調査対象になります。

担当を決める際は、[更新工程ごとの状態と責任の表](https://pki-channel.com/ja/articles/acme-renewal-deployment-design#_1)に、今回のCertificate・Secret・利用先を当てはめてください。どこから先を別の監視や担当へ渡すかが明確になります。

## 更新時の責任を分ける

cert-managerがSecretを更新した後、利用側が変更を取り込む方法は製品によって違います。Secretをファイルとして使うアプリ、Ingressコントローラー、外部ロードバランサーを同じものとして扱わないでください。

引き継ぎには、Certificateの名前空間と名前、Issuerの参照、Secret名、利用するリソース、反映方法、実URLからの確認方法を残します。秘密鍵の中身を台帳へ貼る必要はありません。

`Ready=True` は発行側を調べる重要な証拠です。利用者が受け取る証明書まで確認して初めて、希望した名前でサービスが動いていると判断できます。この二つを分けると、障害時にcert-managerとIngressの間で担当を押し付け合わずに済みます。

## 一次資料

- [cert-manager — Certificate resource](https://cert-manager.io/docs/usage/certificate/)
- [cert-manager — Issuer](https://cert-manager.io/docs/concepts/issuer/)
- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html)
