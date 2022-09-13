# IDaaSの活用に起因する脆弱性の悪用
## 概要

本稿では、ウェブやモバイルのアプリケーションのユーザーへのサインインおよびサインアップ機能を提供するサービスであるIdentity as a Service(以後 IDaaSと表記する)の活用と、それに起因するアプリケーションの脆弱性について解説を行います。

また、エンタープライズ向けのID統合管理サービスについては本稿では触れませんのでその点ご了承いただければと思っております。

## IDaaSとは

IDaaSは大きな区分ではSaaS(Software as a Service)と同じ区分であり、ウェブやモバイルのアプリケーションのユーザーへのサインインおよびサインアップ機能などの認証における機能を提供するクラウドサービスです。

IDaaSの例として以下のようなものが存在します。

- [Amazon Cognito User Pool](https://aws.amazon.com/jp/cognito/)
- [Firebase Authentication](https://firebase.google.com/docs/auth?hl=ja)
- [Google Identity Platform](https://cloud.google.com/identity-platform?hl=ja)
- [Microsoft Azure AD BtoC](https://azure.microsoft.com/ja-jp/services/active-directory/external-identities/b2c/#overview)
- [Auth0](https://auth0.com/jp)

## 活用に起因する脆弱性

ここからは、IDaaSの利用方法が起因する脆弱性について解説を行ってきたいと思います。

### 意図しないサインアップ経路の存在
####  概要

管理者画面や会員向けのサービスなどの設計や要件レベルでサインアップ経路を制限しているアプリケーションにおいて、IDaaSの設定不備や誤った利用によって意図しない形でのサインアップが可能になっている場合が存在します。

#### 原因

管理者画面や会員向けのサービスなどの設計や要件レベルでサインアップ経路の例として、管理者以外がユーザーの作成を想定していないケースが存在します。本稿ではAmazon Cognitoの誤った利用を例に原因について解説を行っていきます。

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

Amazon Cognito User Poolをはじめ、一部のIDaaSではクライアントIDのような識別子を用いて発行されるTokenに権限設定している。Cognitoではアプリクライアントと呼ばれるクライアントIDを用いてTokenを発行しており、このアプリクライアントに設定される権限をOIDCのScopeとして、JWT形式のトークンを発行します。

このアプリクライアントを一般ユーザーと管理者ユーザーで別けて利用している場合、属性へのアクセスに関しては制御可能だが、サインアップに関してはアプリクライアントを介して制御を行うことができないので本来意図したサインアップ経路の制限を行うことはできませんので、第三者がこのアプリクライアントを介して自己サインアップが行えてしまいます。

#### 影響

本来であれば閲覧を想定していない第三者が、該当の画面やエンドポイントへのアクセスを行える可能性があります。

#### 対策

限定されたサインアップの場合、自己サインアップをはじめとした、利用者が自らユーザーを作成できる機能の無効化をおこなってください。また、自己サインアップ機能を無効化できない場合、それに準ずる対策をおこなってください。

また、管理者による作成に際しては、各IDaaSの提供するAPIへのアクセス方法に準拠し管理者用APIを用いて作成を行なっていることを確認してください。

**確認リスト**

[ToDo] フローチャート化する

- [ ] 仕様上、第三者のサインアップを許可していない
- 第三者のサインアップを許可していない場合
  - IDaaSの設定で自己サインアップが制御できる
  	- [ ] 自己サインアップが許可されていない
  - IDaaSの設定で自己サインアップが制御できない
  	- [ ] 他の機能を用いて認可制御を行える

**根本的対策**

自己サインアップを許可しない設計や要件の場合、根本的対策として該当する機能を無効化し、ユーザーによるサインアップを許可しないようにしてください。無効後に新規作成を行う際は、RBAC等のアクセス制御をもちいて、操作が可能なユーザーを絞りユーザーの新規作成を行なってください。

また、IDaaSにおいて、自己サインアップを無効化できない場合は、管理者による新規作成フローにおいて、利用者から操作できないカスタム属性などを用いて、該当アカウントが管理者作成のアカウントであることを証明する属性を追加し、認可やアクセス制御の際に確認を行なってください。

**一時的対策**

一時的対策として、IDaaSへのIP制限等を行い、管理者利用するIDaaSへの一般ユーザーのアクセスをIPレベルで制限してください。

また、本対策を恒久的な対策とせず、根本的な対策を用いて順次対策を行なってください。

**具体: Amazon Cognitoの場合**

Amazon Cognito User Poolにおいては、ユーザーの自己サインアップを無効化する機能が存在するので、この機能の無効化をおこなってください。

管理者用APIへのアクセスに際して、Cognito User PoolのユーザーグループとAmazon Cognito ID Poolを用いて、AWSのCognito User Poolの管理者APIへのアクセスが可能な認証情報を発行してください。この発行される認証情報は最小権限の原則に乗っ取り、ユーザーグループを用いて管理をおこなってください。

また、根本的な対策にはなりませんが、管理者が特定のIPからのみ接続する場合は、AWS WAFを用いたIP制限を行うことで、CognitoのすべてのAPIへのアクセスに関する制限を行うことで一時的な対策を施すことも可能です。

**具体: Google Identity Platform の場合**

Google Identity Platformにおいては、ユーザーの自己サインアップを無効化する機能が存在するので、この機能の無効化をおこなってください。

また、管理者APIへのアクセスに際しては、Admin SDKを用いたAPIを実装し、それを介してアクセスをおこなってください。

**具体: Firebase Authentication の場合**

Firebase Authentication においては、ユーザーの自己サインアップ機能が存在しないため、無効化を行うことはできません。次善的対策として、custom claimによる登録フローに関する属性を追加し、セキュリティルールにおけるバリデーションで不正に作成されたユーザーが操作ができないように確認を行なってください。

また、custom claimに関しては管理者APIへのアクセスへのアクセスが必要なため、Admin SDKを用いたAPIを実装し、それを介してアクセスをおこなってください。

#### 学習方法/参考文献

 - [Amazon Web Service - 管理者作成ユーザーのポリシーの設定](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-settings-admin-create-user-policy.html)
 - [Amazon Cognito における AWS WAF のネイティブサポートが開始](https://aws.amazon.com/jp/about-aws/whats-new/2022/08/amazon-cognito-enables-native-support-aws-waf/)
 - 
 - [Google Cloud - プロジェクトにおける Identity Platform ユーザー > ユーザーのセルフサービス](https://cloud.google.com/identity-platform/docs/concepts-manage-users?hl=ja&_ga=2.183319137.-699514871.1655112464#user-actions)
 - [Flatt Security inc. - Firebase Authentication 7つの落とし穴 - 脆弱性を生むIDaaSの不適切な利用](https://blog.flatt.tech/entry/firebase_authentication_security)
 - [Flatt Security inc. - AWS 診断を事例としたクラウドセキュリティ。サーバーレス環境の不備や見落としがちな Cognito の穴による危険性](https://blog.flatt.tech/entry/cloud_security_aws_case#341-Cognito-User-Pool)
 - [NoSoSecure - Hacking AWS Cognito Misconfigurations](https://notsosecure.com/hacking-aws-cognito-misconfigurations)
 - [個人ブログ - AWS Cognito Misconfigurations in Android Apps](https://erev0s.com/blog/aws-cognito-misconfigurations-in-android-apps/)

### 権限に関するカスタム属性の変更
#### 概要

IDaaSの機能を誤って利用することにより、ユーザーに紐づくカスタム属性などの変更が可能になり、アプリケーションにおける権限昇格や意図しないアクセスを引き起こす可能性が存在します。

#### 原因

権限情報に関するカスタム属性を変更する機能のアクセス制御が適切に行われていないことから、第三者が属性の変更をおこなってしまい権限を書き換える可能性が存在します。本稿ではAmazon Cognitoの誤った利用を例に原因について解説を行っていきます。

**誤った利用の例 1. 権限に関する属性が変更可能**

ユーザーに対してアプリケーションの権限に関する属性の変更権限を与えることにより、属性を任意の値に変更される恐れがあります。

```sh
$ ACCESS_TOKEN="eJy..." # JWT形式のアクセストークン
$ aws cognito-idp update-user-attributes \
  --user-attributes Name="custom:role",Value="admin" \ # アプリケーションの権限に関わるカスタム属性の変更
  --access-token $ACCESS_TOKEN
```

**誤った利用の例 2. 公開されたによるクライアントIDによるアクセス制御**

一部IDaaSでは、自らの情報へアクセスを行う際にクライアントIDのような特定の識別子を用いてアクセスを行います。このクライアントIDはユーザーの属性に関する変更や閲覧に関する操作を司っており、OIDCにおけるScopeのような動作をします。

このクライアントIDを用いてユーザーの属性変更に関するアクセス制御を行なっており、かつ取得に制限のかけられていないJavaSctriptファイルやapkファイルなどにこの管理者用のクライアントIDが含まれている場合、攻撃者はこのクライアントIDを用いてTokenを発行し、自らの権限に関わる属性を変更しアプリケーション内の権限を昇格させることが可能です。

#### 影響

本来であれば閲覧を想定していない第三者が、該当の画面やエンドポイントへのアクセスを行える可能性があります。

#### 対策

**根本的対策**

根本的対策として、一般ユーザーから権限に関わるカスタム属性の値を変更させないようにしてください。権限に関わるカスタム属性の操作を必要とする場合はRBAC等を用いて管理者権限のもののみが変更できるようにしてください。

また、カスタム属性を利用しないで権限を管理する機能がある場合、その機能を用いることも検討してください。

**一時的対策**

一時的対策として、IDaaSへのIP制限等を行い、管理者利用するIDaaSへの一般ユーザーのアクセスをIPレベルで制限してください。

また、本対策を恒久的な対策とせず、根本的な対策を用いて順次対策を行なってください。

**具体: Amazon Cognito User Pool**

Amazon Cognito User Poolにおいては、ユーザーの属性で権限情報を管理することも可能ですが、Cognito User Poolの管理者APIに対してRBACを実装する場合、このユーザーグループを用いて管理することも検討してください。

#### 学習方法/参考文献

- [Amazon Web Service - ユーザープール属性](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-settings-attributes.html)
- [Amazon Web Service - ユーザーグループ](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pools-user-groups.html)
- [Flatt Security inc. - AWS 診断を事例としたクラウドセキュリティ。サーバーレス環境の不備や見落としがちな Cognito の穴による危険性](https://blog.flatt.tech/entry/cloud_security_aws_case#341-Cognito-User-Pool)

### デフォルトエラーによるユーザーの存在判定
#### 概要
- デフォルトのエラーメッセージを有効化していることにより、利用しているユーザーの情報が閲覧できてしまう可能性がある

#### 原因と影響
- デフォルトエラーが詳しく返してしまうのが原因
- ユーザーの存在が暴露される

#### 事例紹介

#### 対策
- 統一されたカスタムメッセージに変更
- 対策が不可能なサービスも存在する(Firebase AuthのID/Pass認証)

#### 学習方法/参考文献

### 過剰な権限の付与
#### 概要
- クラウドサービスへの権限を付与するサービスにおける過剰な権限の付与

#### 原因と影響

#### 事例紹介

#### 対策
- 最小権限の原則の遵守
	- 必要以上の操作権限を付与しない
	- 必要なリソースへのアクセスを許可する

#### 学習方法/参考文献

### EDoS(Economic Denial of Sustainability)
#### 概要
- 課金の要素が増えることによるEDoSが想定される
	- Cognitoの場合、Userの数で課金される
		- 検証していなくても課金tな衣装になる

#### 原因と影響


#### 事例紹介

#### 対策
- 同一IPからのリクエストに対する検知

#### 学習方法/参考文献
