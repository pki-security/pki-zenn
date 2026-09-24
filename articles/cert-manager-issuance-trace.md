---
title: "cert-managerで証明書が発行されないとき、CertificateからChallengeまでどう追うか"
emoji: "🔎"
type: "tech"
topics: ["kubernetes", "certmanager", "acme", "tls", "sre"]
published: true
---

`Certificate`の`READY`が`False`になっているとき、最初に必要なのは「発行に失敗した」という結論より、**どの処理が、何を待っているか**の特定です。

この記事では、ACME Issuerを使う構成について、関連するリソースをたどる調査手順を整理します。2026年9月24日に確認した公式資料を基にしています。Kubernetesクラスターを用いた再現試験は今回行っていません。リソース名は説明用で、実際の名前に置き換えてください。

## 調査の地図を先に置く

発行要求を追う経路と、証明書が使われる経路を並べると、調査範囲が見えます。

```text
発行要求を追う経路（ACME Issuerの場合）
Certificate → CertificateRequest → Order → Challenge
                      │
                      └─ issuerRefでIssuer / ClusterIssuerを確認

発行後に利用状態を確認する経路
Certificate.spec.secretName → Secret → TLS終端 → 利用者からの接続
```

上段のOrderとChallengeはACME用です。CA Issuerなど別方式で発行している場合に、それらがないことだけで異常とは判定できません。下段は実際の構成に合わせて追う運用上の確認経路です。[cert-manager：Troubleshooting](https://cert-manager.io/docs/troubleshooting/)

## 1. 対象のCertificateを固定する

最初にクラスターの接続先、namespace、Certificate名をそろえます。以下は参照コマンドです。`example-app`は対象namespace、`web-tls`は対象Certificateに置き換えます。

```sh
kubectl config current-context
kubectl -n example-app get certificates.cert-manager.io
kubectl -n example-app describe certificate web-tls
```

Certificateがない場合は、作成元を確認します。Ingressから作られる設計なのか、Certificateを直接管理する設計なのかで、調べる設定は異なります。Certificateがない段階でDNSチャレンジを探し続けても、発行処理が始まっていない可能性があります。

Certificateがあるなら、次を記録します。

- `spec.dnsNames`：今回の対象名か。
- `spec.issuerRef`：参照している発行元のname・kind・group。
- `spec.secretName`：最終的な格納先。
- ConditionsとEvents：理由、メッセージ、関連する要求名、観測時刻。

`Ready=False`だけで原因を決めず、イベントに出ているCertificateRequest名を次の手掛かりにします。

## 2. 名前の似た要求ではなく、対象の要求を選ぶ

```sh
kubectl -n example-app get certificaterequests.cert-manager.io
kubectl -n example-app describe certificaterequest REQUEST_NAME
```

`REQUEST_NAME`は、前段のEventsや関連付けを確認して置き換えます。名前の接頭辞や一覧の最後の行だけで選ばず、対象Certificateとの`metadata.ownerReferences`、作成時刻、revisionを照合します。過去の失敗した要求と現在の要求を混ぜないためです。

要求では、発行結果に加えて承認状態を読みます。

| 観測 | 次の確認 |
|---|---|
| `Approved=True` | 承認されたことは分かる。発行の完了はReady等で別に確認 |
| `Denied=True` | 拒否理由と適用された承認ポリシーを確認 |
| Approvedがまだ付いていない | 採用している承認の仕組みと、その処理状況を確認 |
| `Ready=False` / `Pending` | Issuerの準備待ちか、後続処理待ちかをメッセージで分ける |
| `Ready=False` / `Failed` | そのCertificateRequestは失敗状態。現在の後続要求の有無も確認 |

CertificateRequestの`spec`は作成後に変更できません。原因が分からないまま既存要求を書き換える手順にはしません。[cert-manager：CertificateRequest resource](https://cert-manager.io/docs/usage/certificaterequest/)

## 3. Issuerの種類とスコープを合わせる

`issuerRef.kind`が`Issuer`ならnamespace内のIssuer、`ClusterIssuer`ならクラスタースコープのClusterIssuerを確認します。外部Issuerを使う構成では`group`も確認します。

```sh
# issuerRef.kind が Issuer の場合
kubectl -n example-app describe issuer ISSUER_NAME

# issuerRef.kind が ClusterIssuer の場合
kubectl describe clusterissuer CLUSTER_ISSUER_NAME
```

上の2行は使っている種類に応じて選びます。ACMEアカウントの登録や参照設定など、ここで停止しているなら、その問題を先に解決します。Issuerが準備できていることだけで、個々の証明書の認証が成功したとは判断しません。

## 4. ACMEならOrderと、その配下のChallengeへ進む

```sh
kubectl -n example-app get orders.acme.cert-manager.io
kubectl -n example-app describe order ORDER_NAME
kubectl -n example-app get challenges.acme.cert-manager.io
kubectl -n example-app describe challenge CHALLENGE_NAME
```

ここでも、CertificateRequestのEventsとOrder、OrderとChallengeの所有関係を照合します。複数のドメイン名が含まれる場合は、一つのChallengeの結果で全体を代表させず、対象Orderに関係する認証を確認します。

公式資料では、OrderとChallengeを通してACME側の処理を追えます。認証が不要な状態などもあるため、Challengeが見当たらないことだけで失敗とはせず、Orderの状態・認証情報・履歴を読みます。[cert-manager：ACME Orders and Challenges](https://cert-manager.io/docs/concepts/acme-orders-challenges/)

| 止まっている場所 | 見る証拠 | 調査の方向 |
|---|---|---|
| 要求からOrderへ進んでいない | CertificateRequestの条件、Events、Issuer | 承認・発行元・要求のエラー |
| DNS01の提示処理で失敗 | ChallengeのReason、DNSプロバイダーのエラー | 選択したsolver、ゾーン、権限、認証情報の参照先 |
| self-check待ち | ChallengeのReason、実際のsolver設定 | cert-managerからのDNS/HTTP到達経路 |
| CAの検証で失敗 | Order/Challengeの状態とCAのエラー | 対象名、公開側からの到達性、認証内容 |
| Orderは進んだがCertificateが未完了 | CertificateRequestとCertificateの最新Events | 後続処理のどこにエラーがあるか |

DNS01で`Presented=True`でも、認証完了とは限りません。公式のトラブルシュート資料にも、TXT提示後にself-checkを待つ例があります。自己確認の成功とCAの検証結果を別々に読みます。[cert-manager：ACMEトラブルシューティング](https://cert-manager.io/docs/troubleshooting/acme/)

DNS01のself-checkでは、既定の問い合わせ経路と明示したDNS設定を照合します。CNAME委任を使う場合は、その追従設定も調査対象です。共有クラスター全体のDNS設定を、ひとつの認証のエラーだけで変更しないようにします。[cert-manager：DNS01設定](https://cert-manager.io/docs/configuration/acme/dns01/)

## 5. 発行できたら、Secretと配信先の調査へ切り替える

発行された証明書は、Certificateの`spec.secretName`が指す同一namespaceのSecretに保存されます。発行の完了後は、TLS終端がそのSecretを参照しているか、変更を取り込んだかを確認します。[cert-manager：Certificate resource](https://cert-manager.io/docs/usage/certificate/)

```sh
# Secretのメタデータ等を確認する。秘密鍵を含む全内容は出力しない
kubectl -n example-app get secret TARGET_SECRET
```

この一覧だけでは証明書の中身や実際の配信は検証できません。「Secretがある」と「利用者が目的の証明書を受け取る」を分け、必要な権限と手順で証明書の識別情報、参照先、実接続を照合します。秘密鍵を含むSecret全体を障害チケットへ貼り付ける必要はありません。

保存された証明書と配信される証明書の照合へ進む場合は、運営するPKIちゃんねるの[cert-managerの調査コマンド集](https://pki-channel.com/ja/commands/cert-manager-troubleshooting/?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b007-1)を補足として使えます。

## 引き継ぎは「最後の成功」と「最初の未完了」を一組にする

以下は障害引き継ぎ用の記入枠です。状態名は実際に観測した値を入れ、分からない欄は未確認とします。

```text
観測時刻・タイムゾーン：
context / namespace / Certificate：
cert-managerのバージョン：
対象CertificateRequest / Order / Challenge：
関連付けを確認した根拠：
最後に確認できた処理：
最初の未完了処理とReason：
変更した条件と、その後の観測：
利用者側での配信確認：未確認 / 確認済み（条件と証拠）
次に調べる担当・期限：
```

リソースを削除して作り直す前に、この情報を残すと、同じ失敗を繰り返したのか、新しい段階へ進んだのかを区別できます。再発防止として発行・配置・配信確認の責任を見直す場合は、PKIちゃんねるの[更新運用の責任分界と完了条件](https://pki-channel.com/ja/articles/acme-renewal-deployment-design?utm_source=zenn.dev&utm_medium=referral&utm_campaign=owned_media&utm_content=own-b007-2)にも整理しています。

調査結果は「cert-managerが壊れた」で終わらせず、**対象の要求がどのリソースで止まり、次に確認する証拠は何か**まで記録すると、対応を引き継ぎやすくなります。

---

調査・執筆にAIを使用しています。公式資料の確認日は2026年9月24日です。本稿のコマンドは調査手順の例で、今回の実機試験結果ではありません。採用バージョンと構成に合わせて確認してください。
