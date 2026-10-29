---
title: "cert-managerのIssuerとClusterIssuer：名前だけで参照しない"
emoji: "🔎"
type: "tech"
topics: ["security", "tls"]
published: true
---

Issuerを作ったのにCertificateから使えない。ここでCertificateは証明書の取得・管理を指示するcert-managerのリソース、namespaceはKubernetes内でリソースを区分する名前空間です。こうした調査では、名前が一致しているかだけでなく、種類とnamespaceを一緒に読みます。IssuerとClusterIssuerの違いは、発行方式の強さではなく、主にリソースの参照範囲です。

![Issuer: 同じnamespaceのCertificateから利用する発行設定。 ClusterIssuer: 複数namespaceから参照できるクラスタ共通の発行設定。 運用の確認: 認証情報の場所と、変更の影響範囲を別に確認する。](/images/b077-issuer-scope.png)

説明用の概念図。実測結果ではありません。

## Issuerはnamespaceに属する

Issuerはnamespaceを持つリソースで、基本的に同じnamespaceのCertificateから利用します。別のnamespaceに同名のIssuerがあっても、同じリソースではありません。

障害メモに名前だけを書くと、似た設定を見比べて迷います。「kind／namespace／name」を一組で記録します。

## ClusterIssuerはクラスタ全体のリソース

ClusterIssuerはクラスタスコープで、複数のnamespaceから参照できます。ただし、クラスタ全体から参照できることと、誰でも任意の証明書を発行してよいことは別です。

発行の許可、利用するCA、認証情報の権限などは、クラスタの運用設計として確認します。

## issuerRefで意図を明示する

Certificateの`issuerRef`には、名前だけでなく、意図したkindやgroupを確認します。参照先を変えるときは、対象の証明書をどのCAで管理するのかまで追います。

Secretの置き場所も、IssuerとClusterIssuerで同じ思い込みをしないでください。ClusterIssuerの認証情報などは設定されたクラスタリソースnamespaceの扱いを公式手順で確認します。

## 共有する前に責任者を決める

一つのClusterIssuerを複数チームで使うと、変更の影響も広がります。アカウント鍵やCA設定を変える担当、異常を受ける窓口、変更前の確認対象を決めます。

最初の学習用メモは「発行するCertificateのnamespace／参照するkindとname／認証情報の管理場所／変更責任者」。これだけでも、設定のコピーを超えて構成を読めるようになります。

## 一次資料

- [cert-manager: Issuer](https://cert-manager.io/docs/concepts/issuer/)
- [cert-manager: ACME configuration](https://cert-manager.io/docs/configuration/acme/)

## もう少し詳しく知りたい方へ

本文で扱った仕組みや運用を、図解や関連する確認項目まで掘り下げる資料です。

- [GitOpsで進める証明書運用](https://pki-channel.com/ja/articles/gitops-certificates?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b077-1)
- [証明書プロファイル設計](https://pki-channel.com/ja/articles/certificate-profile-design?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b077-2)

---

資料確認・執筆・図解制作にAIを使用しています。例や確認表は説明用で、実機試験の結果ではありません。
