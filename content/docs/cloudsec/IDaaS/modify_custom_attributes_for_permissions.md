---
title: アプリケーションの権限に関するカスタム属性の変更 - IDaaS
weight: 1

---

# アプリケーションの権限に関するカスタム属性の変更
## 概要

IDaaSの機能を誤って利用することにより、ユーザーに紐づくカスタム属性などの変更が可能になり、アプリケーションにおける権限昇格や意図しないアクセスを引き起こす可能性が存在します。

## 原因

権限情報に関するカスタム属性を変更する機能のアクセス制御が適切に行われていないことから、第三者が属性を変更し権限を書き換える可能性が存在します。本稿ではAmazon Cognitoの誤った利用を例に原因について解説します。

**誤った利用の例 1. 権限に関する属性が変更可能**

ユーザーに対してアプリケーションの権限に関する属性の変更権限を与えることにより、属性を任意の値に変更される恐れがあります。

```sh
$ ACCESS_TOKEN="eJy..." # JWT形式のアクセストークン
$ aws cognito-idp update-user-attributes \
  --user-attributes Name="custom:role",Value="admin" \ # アプリケーションの権限に関わるカスタム属性の変更
  --access-token $ACCESS_TOKEN
```

**誤った利用の例 2. 公開されたクライアントIDによるアクセス制御**

一部IDaaSでは、自らの情報へアクセスを行う際にクライアントIDのような特定の識別子を用いてアクセスを行います。このクライアントIDはユーザの属性に関する変更や閲覧に関する操作を司っており、OIDCにおけるScopeのような動作をします。

このクライアントIDを用いてユーザの属性変更に関するアクセス制御をしており、かつ取得に制限のかけられていないJavaSctriptファイルやapkファイルなどにこの管理者用のクライアントIDが含まれている場合、攻撃者はこのクライアントIDを用いてTokenを発行し、自らの権限に関わる属性を変更しアプリケーション内の権限を昇格させることが可能です。

## 影響

本来であれば閲覧を想定していない第三者が、該当の画面やエンドポイントへアクセスできる可能性があります。

## 対策

### 根本的対策

根本的対策として、一般ユーザーから権限に関わるカスタム属性の値を変更させないようにしてください。権限に関わるカスタム属性の操作を必要とする場合はRBAC等を用いて管理者権限のユーザーのみが変更できるようにしてください。

また、カスタム属性を利用しないで権限を管理する機能がある場合、その機能を用いることも検討してください。

### 一時的対策

一時的対策として、IDaaSへのIP制限等を行い、管理者利用するIDaaSへの一般ユーザーのアクセスをNWレベルで制限してください。

また、本対策を恒久的な対策とせず、根本的な対策を実施してください。

### 具体: Amazon Cognito User Pool

Amazon Cognito User Poolにおいては、ユーザーの属性で権限情報を管理可能ですが、Cognito User Poolの管理者APIに対してRBACを実装する場合、このユーザグループを用いて管理することも検討してください。

## 学習方法/参考文献

- [Amazon Web Service - Amazon Cognito - ユーザープール属性](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-settings-attributes.html)
- [Amazon Web Service - Amazon Cognito - ユーザーグループ](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pools-user-groups.html)
- [Flatt Security inc. - AWS 診断を事例としたクラウドセキュリティ。サーバーレス環境の不備や見落としがちな Cognito の穴による危険性](https://blog.flatt.tech/entry/cloud_security_aws_case#341-Cognito-User-Pool)
