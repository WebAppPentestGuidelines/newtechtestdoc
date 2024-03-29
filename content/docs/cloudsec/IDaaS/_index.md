---
title: IDaaSの活用に起因する脆弱性とその悪用
weight: 1

---
<!-- textlint-disable ja-technical-writing/sentence-length -->

# IDaaSの活用に起因する脆弱性とその悪用
## 概要

本稿では、Webやモバイルのアプリケーションのユーザーへのサインインおよびサインアップ機能を提供するサービスであるB2C向けのIdentity as a Service(以後 IDaaSと表記する)の活用に起因するアプリケーションの脆弱性について解説します。

## IDaaSとは

IDaaSは大きな区分ではSaaS(Software as a Service)と同じ区分であり、Webやモバイルのアプリケーションのユーザーへのサインインおよびサインアップ機能、ユーザの管理機能などの認証やそれに関わる機能を提供するクラウドサービスです。

上記のような用途で利用されるIDaaSの例として以下のようなものが存在します。

- [Amazon Cognito User Pool](https://aws.amazon.com/jp/cognito/)
- [Firebase Authentication](https://firebase.google.com/docs/auth?hl=ja)
- [Google Identity Platform](https://cloud.google.com/identity-platform?hl=ja)
- [Microsoft Azure AD BtoC](https://azure.microsoft.com/ja-jp/services/active-directory/external-identities/b2c/#overview)
- [Auth0](https://auth0.com/jp)

## 活用に起因する脆弱性

ここからは、IDaaSの活用に起因する脅威 / 脆弱性について解説します。

IDaaSを用いる際に発生しうる脅威 / 脆弱性は大きく分けて3つに分類できます。

1. 仕様として定義していないアクションの存在が悪性の副作用をもたらすもの
2. 仕様として定義したものが適切にハンドリングされず悪性の副作用をもたらすもの
3. IDaaSから取得した値やTokenの対に完全な信頼をおくことによる、値の悪性作用をもらすもの

本解説では、上記の分類に該当する脆弱性について記述していきます。

### 脆弱性 / 脅威 / 悪用

- [意図しないサインアップ経路の存在](./presence_of_unintended_signup_paths)
- [権限に関するカスタム属性の変更](./modify_custom_attributes_for_permissions)
- [デフォルトエラーによるユーザーの存在判定](./default_error)
- [EDoS(経済的/資金的なサービス停止攻撃)](./edos)
