---
title: クラウドストレージサービスにおける設定不備
weight: 999
---

# クラウドストレージサービスにおける設定不備

## 概要

クラウドストレージサービスとは、インターネット上にてファイルの保管・共有などができるサービスです。
画像やJavaScriptファイルなどの静的ファイルを公開する用途などでも利用されています。
有名処のものとしては以下のようなサービスがあります。

* [AWS S3](https://aws.amazon.com/s3/)
* [Azure Blob Storage](https://azure.microsoft.com/services/storage/blobs/)
* [Cloud Storage](https://cloud.google.com/storage)

近年ではこれらクラウドストレージサービスの設定ミスによるセキュリティインシデントが発生しています。

## 原因と影響

クラウドストレージサービスにおけるアクセス許可の設定不備が原因となり、意図しない情報がインターネット上に公開されてしまう可能性があります。
また、ファイルのアップロードが可能となっている場合には、公開しているコンテンツに悪意あるコードを埋め込まれ、攻撃に利用されてしまう可能性もあります。

## 診断観点

クラウドストレージの設定ミスを確認するシンプルな方法としては、クラウドストレージのエンドポイントに対応するURLへアクセスすることです。
そして、セキュリティ上問題となるような設定やコンテンツが公開されていないかを確認します。
本記事ではAWS S3を例として、脆弱性診断における本手法について説明します。

### S3バケットを示すエンドポイントのURLについて調査

クラウドストレージを示すURLの形式に関する仕様は以下のように公開されています。

* [AWS S3](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/access-bucket-intro.html)
* [Azure Blob Storage](https://learn.microsoft.com/ja-jp/azure/storage/blobs/storage-blobs-introduction)
* [Cloud Storage](https://cloud.google.com/storage/docs/request-endpoints?hl=ja)

ここではAWS S3について解説しますが、S3ではクラウドストレージのエンドポイントを示すURLは以下2種類のタイプが存在します。

* 仮想ホスト形式
    1. `http://<backet-name>.s3-<region>.amazonaws.com/`
    2. `http://<backet-name>.s3.amazonaws.com/`  
    2019 年 3 月 20 日より後に開始されたリージョンで作成されたバケットでは2の形式(レガシーグローバルエンドポイント)では到達できないようです。

* パス形式
    1. `http://s3-<region>.amazonaws.com/<backet-name>`
    2. `http://s3.amazonaws.com/<backet-name>`  
    米国東部 (バージニア北部) リージョンは2の形式で、ほかのリージョンは1の形式となります。パス形式は将来的には廃止が検討されているようです。

上記の仕様を利用して、特定キーワードなどに関連するS3バケットのエンドポイントのURLを推測できます。
また、S3バケットにおいて独自ドメインを利用して、静的なサイトをホスティングする場合には、前提としてバケット名とドメイン名が一致しなければなりません。
このため、Webページのドメイン名などからもエンドポイントのURLを推測できます。

特定のキーワードに関連するS3バケットを探索するようなツールも多数公開されています。

* [s3recon](https://github.com/clarketm/s3recon)
* [s3finder](https://github.com/magisterquis/s3finder)

また、公開状態のクラウドストレージやファイルの情報を定期的に収集し、検索できるオンラインサービスなども存在します。本サービスでは、S3だけではなくAzureやGCPなど他クラウドサービスの情報も検索できます。

* [Grayhatwarfare](https://grayhatwarfare.com/)

脆弱性診断の実施時には、HTMLソース・HTTPレスポンス・モバイルアプリケーションのソースコード内などに、S3バケットのURLや探索するためのキーワードが、存在しないかを確認すべきでしょう。
発見したキーワードをもとにクラウドストレージのエンドポイントを探索してみてください。
なお、探索によって存在が判明したS3バケットのエンドポイントのURLが診断対象の範囲に含まれるかどうかについては、調査する前にあらかじめ顧客やサイト管理者に対して確認するべきであると考えます。

### ブラウザでの確認

ブラウザ経由でアクセスして、S3バケットにおいてファイルの一覧の表示が許可されている場合には以下のようなXML形式のレスポンスが表示されます。

![ブラウザで直接アクセス](../image/s3.png)

この設定状態の場合には、不特定多数へストレージ内に存在するファイルの内容が判明してしまうため、いわゆるディレクトリリスティングと同様の問題があるといえます。そのためこの設定状態が意図したものであるかを確認すべきでしょう。

### コマンドラインでの確認

コマンドラインを用いて、クラウドストレージのファイルのダウンロードや、ファイルのアップロードについて確認できます。

S3バケットであれば[AWS CLI](https://aws.amazon.com/jp/cli/)を利用することで上記をテストできます。
S3においては、S3バケットのACLが`All Authenticated AWS Users`と設定されている可能性があります。
そのため、AWSユーザーとして認証された状態でもコマンドラインを実行する方が良いでしょう。

* S3バケットの内容を表示
```
aws s3 ls s3://<backet-name>/<path>
```

* S3バケットのオブジェクトを取得
```
aws s3 cp s3://<backet-name>/<objectname> <output file name>
```
```
aws s3api get-object --bucket <bucket-name> --key <key name> <output file name>
```

設定ミスによって書き込みが可能となっている可能性があるため、以下のコマンドを利用してファイルの書き込みが可能であるかを確認できます。
正し、脆弱性診断においては、書き込みをテストしてよいか事前に顧客側へ確認するべきであると考えます。

* ローカルのファイルを指定して、指定したS3バケットにアップロード
```
aws s3 cp <local file-path> s3://<bucket-name>/<path>
```
```
aws s3api put-object --bucket <bucket-name> --key <key name> --body <local file-path>
```

## 事例紹介

* S3の設定不備に関するHacker Oneのレポート   
https://hackerone.com/reports/128088  
https://hackerone.com/reports/1062803  
https://hackerone.com/reports/129381

* S3設定不備により機密データが公開されていた事例  
https://www.skyhighsecurity.com/en-us/about/resources/intelligence-digest/unsecured-servers-can-put-lives-at-stake.html  
https://www.safetydetectives.com/news/doctorsme-leak-report/

* Azure Blob Storageの設定不備により機密データが公開されていた事例  
https://www.vpnmentor.com/blog/report-microsoft-dynamics-leak/
https://www.theregister.com/2020/12/01/investment_fund_data_breach/  
https://www.techradar.com/news/microsoft-azure-breach-left-thousands-of-customer-records-exposed  
https://www.bleepingcomputer.com/news/security/exposed-azure-bucket-leaked-passports-ids-of-volleyball-reporters/

* Cloud Storageの設定不備により機密データが公開されていた事例  
https://www.comparitech.com/blog/information-security/google-cloud-buckets-unauthorized-access-report/

* ファイルアップロードが可能となっているような設定ミスを攻撃者に悪用された事例  
https://japan.zdnet.com/article/35139832/

## 対策

公開を意図しないクラウドストレージが存在しないように、アクセス制御の設定を適切に行うようにしてください。
また、公開を意図するクラウドストレージでは、機微な情報が含まれているファイルが存在しないか確認してください。

## 学習方法/参考文献

* [Amazon S3の脆弱な利用によるセキュリティリスクと対策](https://blog.flatt.tech/entry/s3_security)  
上記は本記事にて解説しているアクセス制限の設定不備を含めた、S3にまつわる脆弱性について詳細に解説したブログ記事です。本記事では取り扱っていないWebアプリケーションでS3の署名付きURLを生成している場合に脆弱性が生じてしまう事例やS3におけるサブドメインの乗っ取りなどについても解説されています。

* [Hunting Azure Blobs Exposes Millions of Sensitive File](https://www.cyberark.com/resources/threat-research-blog/hunting-azure-blobs-exposes-millions-of-sensitive-files)  
Azure Blob Storage設定不備の調査方法に関する記事です。

