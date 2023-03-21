---
title: デフォルトエラーに起因するユーザーの開示 - IDaaS
weight: 1

---

# デフォルトエラーに起因するユーザーの開示
## 概要

攻撃者にとってアプリケーションが返すエラーレスポンスは、攻撃を実施する上で大きなヒントを与えることがあります。代表的な例として、Internal Server Error時にサーバー上のプログラムの断片や、SQL文などが見えてしまうなどがあるでしょう。

認証においても同様で、認証情報がそのサービスで使われているのかを示す詳細なエラーを返すことにより、攻撃者に標的となるユーザーの存在を知らせてしまう可能性があります。

そのような観点からIDaaSでも攻撃者を利するエラーメッセージについては削除や表記の統一を図ることが必要です。

## 原因

CognitoをはじめとするIDaaSのエラーメッセージではデフォルトのメッセージが詳細な情報をレスポンスとして返すケースが存在します。

- [Auth0の場合](https://auth0.com/docs/libraries/common-auth0-library-authentication-errors)
- [Cognitoの場合](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pool-managing-errors.html)

## 影響

エラーメッセージにおいてサービスに登録しているユーザーのメールアドレスを確認することが可能です。
攻撃者はメールアドレスを確認することにより、過去に流出した認証情報との組み合わせをもとにアカウントへの不正アクセス行われる可能性があります。

## 対策

対策として、デフォルトのエラーメッセージを変更し、認証認可に係るエラーに関して統一されたカスタムメッセージに変更することをお勧めします。

また、上記の対策が不可能なサービス(Firebase AuthのID/Pass認証)も存在するため、その点を考慮して対策等をおこなってください。

## 学習方法/参考文献

- [Amazon Web Service - Amazon Cognito - エラーレスポンスの管理](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pool-managing-errors.html)
- [Auth0](https://auth0.com/docs/customize/universal-login-pages/custom-error-pages)
- [OWASP チートシート - 認証とエラー メッセージ](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html#authentication-and-error-messages)
- [Amazon Web Service - Amazon Cognito - 認証チャレンジの作成の Lambda トリガー](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-lambda-create-auth-challenge.html)