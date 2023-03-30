---
title: EDoS(Economic Denial of Sustainability) - IDaaS
weight: 1

---
<!-- textlint-disable ja-technical-writing/sentence-length -->

# EDoS(Economic Denial of Sustainability)
## 概要

IDaaSをはじめとしたクラウドサービスでは、そのサービスの利用に応じて課金される「従量課金制」を採用しています。
このような従量課金を採用するサービスでは、課金される要素が増加するごとに利用額も増加します。
この特性を攻撃者が悪用することで、サービス提供者がサービスを継続できないレベルまで利用額を増加させる攻撃を、Economic Denial of Sustainability(以下 EDoSと呼称する)と呼びます。

本稿ではAmazon Cognitoを例に、このEDoSについて解説をしていきます。

## 原因

「課金される要素が増加する」ことがEDoSを引き起こす原因となります。
Amazon Cognito User Poolでは、外的要因でこの課金額を増加させる方法として、「月間のアクティブユーザー(MAU)の増加」が挙げられます。

このMAUはCognito User Poolに対して、サインインやサインアップ等の機能を利用したユーザーを1として加算され、100,000MAU以下の場合、1MAUあたり0.0055USD加算されます。

MAU|MAUあたりの料金
:---:|:---:
～ 100,000|0.0055USD
90万|0.0046USD
900万|0.00325USD
1,000万超|0.0025USD

このような料金形態を取るCognito User Poolでは、Cognito User Poolのセルフサインアップが有効化されている際に、攻撃者が大量のユーザーを作成することでサービス提供者が意図しない形でMAUが増加し、料金として請求されます。

## 影響

意図しない利用料金の請求により、サービス提供を行うための予算が利用され、サービスを提供できなくなる恐れがあります。

## 事例紹介

現時点では現実世界においてCognito User Poolをはじめとして、IDaaSを用いたEDoSや利用料金の増加は報告されていません。ただし、[NHK厚生文化事業団が運営する寄付サイトへの偽計業務妨害事案](https://www.yomiuri.co.jp/national/20220628-OYT1T50078/)をはじめとして、意図しない課金のためサービスが停止した例や実装の不備により異常な課金が発生した例は報告されています。

## 対策

対策としては、攻撃者のユーザー作成やログインの自動化を難しくすることで大量のユーザーの追加を防ぐことができるでしょう。

自動化を難しくする方法として、[Custom authentication challenge](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-challenge.html)やAWS WAF等のCognito User Poolの拡張機能を用い、reCaptchaの導入や、Lambdaトリガを用いたEmailアドレスの検証とエイリアス利用の制限等が挙げられます。

#### 学習方法/参考文献
- [EDoS Attack: クラウド利用料金でサービスを止められるって本当？: Amazon Cognito User Pool](https://blog.flatt.tech/entry/edos_aws#Amazon-Cognito-User-Pool)
