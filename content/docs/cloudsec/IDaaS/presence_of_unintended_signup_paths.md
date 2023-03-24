---
title: 意図しないサインアップ経路の存在 - IDaaS
weight: 1

---
# 意図しないサインアップ経路の存在
##  概要

管理者画面や会員向けのサービスなどの設計や要件レベルでサインアップ経路を制限しているアプリケーションにおいて、IDaaSの設定不備や誤った利用によって意図しない形でのサインアップが可能になる場合があります。

## 原因

管理者画面や会員向けのサービスなどの設計や要件レベルでのサインアップ経路の例として、管理者以外のユーザによるユーザ作成を想定していないケースが存在します。本稿ではAmazon Cognitoの誤った利用を例に原因について解説します。

```sh
$ CLIENT_ID="xxxxxxxxxxxxxxxxxxxxxxxxxx"
$ USER_NAME="username"
$ PASSWORD="P4ssW0rD"
$ ATTRIBUTES="[]"
$ aws cognito-idp sign-up \
  --client-id $CLIENT_ID \
  --username $USER_NAME \
  --password  $PASSWORD \
  --user-attributes $ATTRIBUTES \
  --no-sign-request
```

**誤った利用の例. サインアップに関する不適切な権限の分離**

Amazon Cognito User Poolをはじめ、一部のIDaaSではクライアントIDのような識別子を用いて発行されるTokenに権限設定しています。Cognitoではアプリクライアントと呼ばれるクライアントIDを用いてTokenを発行しており、このアプリクライアントに設定される権限をOIDCのScopeとして、JWT形式のトークンを発行します。

このアプリクライアントを一般ユーザと管理者ユーザで分けて利用している場合、属性へのアクセスに関しては制御可能ですが、サインアップに関してはアプリクライアントを介して制御できません。そのため、本来意図したサインアップ経路を制限できないため、第三者がこのアプリクライアントを介して自己サインアップが行えてしまいます。

## 影響

本来であれば閲覧を想定していない第三者が、該当の画面やエンドポイントへのアクセスを行える可能性があります。

## 対策

限定されたサインアップの場合、自己サインアップをはじめとした、利用者が自らユーザを作成できる機能の無効化してください。また、自己サインアップ機能を無効化できない場合、それに準ずる対策が必要となります。

また、管理者による作成に際しては、各IDaaSの提供するAPIへのアクセス方法に準拠し管理者用APIを用いて作成していることを確認してください。

### 確認リスト

- [ ] 仕様上、第三者のサインアップを許可していない
- 第三者のサインアップを許可していない場合
  - IDaaSの設定で自己サインアップが制御できる
  	- [ ] 自己サインアップが許可されていない
  - IDaaSの設定で自己サインアップが制御できない
  	- [ ] 他の機能を用いて認可制御を行える

### 根本的対策

自己サインアップを許可しない設計や要件の場合、根本的対策として該当する機能を無効化し、ユーザによるサインアップを許可しないようにしてください。
無効後に新規作成する際は、RBAC等のアクセス制御を用いて、操作が可能なユーザを限定してください。

自己サインアップを無効化できない場合は、管理者による新規作成フローにおいて、利用者から操作できないカスタム属性などを用いて該当アカウントが管理者作成のアカウントであることを証明する属性を追加し、認可やアクセス制御の際に確認してください。

### 一時的対策

一時的対策として、IDaaSへのIP制限等を行い、管理者利用するIDaaSへの一般ユーザーのアクセスをIPレベルで制限してください。

また、本対策を恒久的な対策とせず、根本的な対策を用いて順次対策してください。

#### 具体: Amazon Cognitoの場合

Amazon Cognito User Poolにおいては、ユーザーの自己サインアップを無効化する機能が存在するので、この機能を無効化してください。

管理者用APIへのアクセスに際して、Cognito User PoolのユーザーグループとAmazon Cognito ID Poolを用いて、AWSのCognito User Poolの管理者APIへのアクセスが可能な認証情報を発行してください。この発行される認証情報は最小権限の原則にのっとり、ユーザーグループを用いて管理してください。

根本的な対策にはなりませんが、管理者が特定のIPからのみ接続する場合は、AWS WAFを用いた接続元IP制限によってCognitoのすべてのAPIへのアクセスを制限するという、一時的な対策も可能です。

#### 具体: Google Identity Platform の場合

Google Identity Platformにおいては、ユーザーの自己サインアップを無効化する機能が存在するので、この機能を用いて無効化してください。

また、管理者APIへのアクセスに際しては、Admin SDKを用いたAPIを実装し、それを介してアクセスしてください。

#### 具体: Firebase Authentication の場合

Firebase Authentication においては、ユーザーの自己サインアップ機能が存在しないため、無効化できません。暫定的な対策として、custom claimによる登録フローに関する属性を追加し、セキュリティルールにおけるバリデーションで不正に作成されたユーザが操作できないよう設定してください。

また、custom claimに関しては管理者APIへのアクセスが必要なため、Admin SDKを用いたAPIを実装し、それを介してアクセスしてください。

## 事例
- [Internet-Scale analysis of AWS Cognito Security](https://andresriancho.com/wp-content/uploads/2019/06/whitepaper-internet-scale-analysis-of-aws-cognito-security.pdf)

## 学習方法/参考文献

 - [Amazon Web Service - Amazon Cognito - 管理者作成ユーザーのポリシーの設定](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-settings-admin-create-user-policy.html)
 - [Amazon Cognito における AWS WAF のネイティブサポートが開始](https://aws.amazon.com/jp/about-aws/whats-new/2022/08/amazon-cognito-enables-native-support-aws-waf/)
 - [Google Cloud - プロジェクトにおける Identity Platform ユーザー > ユーザーのセルフサービス](https://cloud.google.com/identity-platform/docs/concepts-manage-users?hl=ja&_ga=2.183319137.-699514871.1655112464#user-actions)
 - [Flatt Security inc. - Firebase Authentication 7つの落とし穴 - 脆弱性を生むIDaaSの不適切な利用](https://blog.flatt.tech/entry/firebase_authentication_security)
 - [Flatt Security inc. - AWS 診断を事例としたクラウドセキュリティ。サーバーレス環境の不備や見落としがちな Cognito の穴による危険性](https://blog.flatt.tech/entry/cloud_security_aws_case#341-Cognito-User-Pool)
 - [NoSoSecure - Hacking AWS Cognito Misconfigurations](https://notsosecure.com/hacking-aws-cognito-misconfigurations)
 - [個人ブログ - AWS Cognito Misconfigurations in Android Apps](https://erev0s.com/blog/aws-cognito-misconfigurations-in-android-apps/)
