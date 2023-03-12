---
title: Webアプリケーションの脆弱性を利用した認証情報の窃取
weight: 999
---

# Webアプリケーションの脆弱性を利用した認証情報の窃取

## 概要

クラウド環境を利用してWebサービスを提供するシステムにおいて、脆弱性を利用されクラウドサービス側の認証情報（認証キー・APIトークンなど）を窃取されてしまう可能性が危険視されています。
これらの認証情報が漏洩した場合には、漏洩した認証情報に紐づく権限を利用され、クラウドプラットフォーム側の機能を悪用されてしまう危険性があります。
上記のような攻撃には従来から存在するWebアプリケーションの脆弱性を利用される可能性があります。

## 原因と影響

認証情報の種類や存在する場所などについては、利用しているクラウドプラットフォームやサービスの種類によって異なります。
あくまで一例ですがシステム内の環境変数・設定ファイルなどにこれらの認証情報が設定されているケースが多く、Webアプリケーションの脆弱性を利用されることによって窃取されてしまう危険性があります。
* Webアプリケーションにて詳細情報を表示するエラー画面などから環境変数に設定された認証情報が漏洩
* ディレクトリトラバーサルの脆弱性を利用されて、設定ファイルに記載された認証情報が漏洩

また、AWSなどのパブリッククラウドサービスではメタデータサービスと呼ばれる機能が存在します。
この機能では、メタデータと呼ばれるインスタンス固有のデータ(インスタンスID、OSデータ、一時的に付与される認証情報など)を内部Web APIを通じて取得するといったしくみが提供されています。
SSRF(Server Side Request Forgery)の脆弱性を利用することで、この機能を悪用してメタデータを不正に取得され、認証情報を窃取されてしまう攻撃手法が知られています。

## 診断観点

本記事ではAWS環境を題材にSSRFの脆弱性を利用して、認証情報を取得する流れについて紹介します。

SSRFは、攻撃者が指定した任意のサーバに対してHTTPリクエストを行うように、サーバサイドアプリケーションを誘導させるというものです。
これを利用して、攻撃者は脆弱性の存在するサーバを踏み台として他サーバにアクセスしたり、自身が直接到達できないサーバに対してアクセスできます。

脆弱性の例として、以下のPHPのサンプルコードを用意しました。

```
<?php
if (isset($_GET['url'])){
    $url = $_GET['url'];
    $conn = curl_init();
    curl_setopt($conn, CURLOPT_URL, $url);
    curl_setopt($conn, CURLOPT_RETURNTRANSFER, true);
    $res =  curl_exec($conn);
    echo $res;
    curl_close($conn);
}
?>
```

GETパラメータで取得したURL値に対して、PHPのcURL関数によりHTTPリクエストを送信し、結果を出力するだけのシンプルなコードです。
もしこのようなWebアプリケーションが存在し、公開されていた場合にはSSRF攻撃を受けてしまう可能性があります。
上記で説明したように、このWebアプリケーションではurlパラメータに指定した値をそのままcURL関数にて取得する実装となっています。
そのため、クライアント側がurlパラメータに任意のURL値を指定して受け渡すことで、Webアプリケーションに指定したURLの対象に対してアクセスさせ、レスポンスを取得できます。

この挙動を利用することで、以下の図のように攻撃者は本来アクセスできないような対象に対して、Webアプリケーションを経由してアクセスを行い、情報を取得できます。
これは、SSRFを用いた内部ネットワークへの攻撃の一例となります。

![SSRFを用いた内部ネットワークへの攻撃](../image/ssrf1.png)

もし、クラウド環境にて提供されているWebアプリケーションに上記のようなSSRFの脆弱性が存在する場合には、この機能を悪用され、メタデータを不正に取得されてしまう可能性があります。以下の図では、AWS環境におけるSSRF攻撃を記載しています。

![SSRFを用いた認証情報の窃取](../image/ssrf2.png)

図ではIAMロールが紐づいた状態のEC2インスタンスにおけるSSRF攻撃によって、Webアプリケーション側がメタデータサーバのAPIへ一時的な認証情報を要求するリクエストを強制させられています。
その結果、EC２のIAMロールに紐づいた認証情報を窃取されてしまう流れを記載しています。
ここでは、EC2にアタッチされているIAMロールに関連付けられた認証情報を、この図のようなSSRF攻撃によって取得する流れの詳細を説明します。

AWSのメタデータサーバに関する仕様は以下に記載されています。

 * [インスタンスメタデータとユーザーデータ](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)

EC2インスタンス上より上記仕様に記載されているURLにアクセスすることで、該当するメタデータの情報を取得できます。
EC2インスタンスにアタッチされたIAMロールが存在する場合には、以下のメタデータサーバのURLにアクセスすることで、認証情報を取得できます。

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/[role-name]
```

上記のパス中の[role-name]の部分にはEC2にアタッチされているIAMロールの名前を指定する必要があります。
上記URLにアクセスすることで指定したIAMロールに関連付けられた一時的なセキュリティ認証情報が格納される仕様となっています。なお、IAMロール名は以下のURLアクセスすることで取得できます。

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```
AWSのEC2上でSSRFの脆弱性が存在するサンプルコードのWebアプリケーション(ssrf.php)が動作しているとします。
まず、SSRFの脆弱性を利用してメタデータサーバのURLをパラメータ値として脆弱性の存在するWebアプリケーションに送信することによって、EC2にアタッチされたrole-nameを取得します。

* リクエストURL
```
ssrf.php?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
```
* レスポンス
```
HTTP/1.1 200 OK
Date: Wed, 22 Feb 2023 17:22:20 GMT
Server: Apache/2.4.55 () PHP/5.4.16
Upgrade: h2,h2c
Connection: Upgrade, close
X-Powered-By: PHP/5.4.16
Content-Length: 7
Content-Type: text/html; charset=UTF-8

ec2role
```

そして、取得したrole-nameを利用してパスの一部を変更したURLの値を送信します。
同様のSSRFの脆弱性を利用することで、一時的に発行された認証情報を取得できます。

* リクエストURL
```
ssrf.php?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2role
```
* レスポンス
```
HTTP/1.1 200 OK
Date: Wed, 22 Feb 2023 17:23:48 GMT
Server: Apache/2.4.55 () PHP/5.4.16
Upgrade: h2,h2c
Connection: Upgrade, close
X-Powered-By: PHP/5.4.16
Content-Length: 1450
Content-Type: text/html; charset=UTF-8

{
  "Code" : "Success",
  "LastUpdated" : "2023-02-22T17:22:41Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIAXXXXXXXXXXXXXXXX",
  "SecretAccessKey" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "Token" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "Expiration" : "2023-02-22T23:56:46Z"
}
```
※認証情報の部分は伏字としています。

このようにして、SSRF攻撃によって認証情報を取得されてしまった場合には、取得された認証情報に紐づく権限を悪用され、被害が拡大してしまう可能性があります。
また、上記のサンプルコードのように送信したURL値の処理に起因するような脆弱性ではなくとも同じ攻撃は成立します。
たとえば以下のようなほかの脆弱性が存在した場合には、その脆弱性を利用することによってアプリケーションに任意のURLにアクセスを行わせるようなSSRF攻撃ができるでしょう。
その結果、同様に認証情報を窃取できる可能性があります。
 * XXE
 * OSコマンドインジェクション

なお、上記はあくまで一例です。
本記事にては解説したように、クラウド環境のWebサービスでは、Webアプリケーションの脆弱性を利用されてクラウド環境側の認証情報の取得などにつながってしまう可能性があります。
そのため脆弱性診断では、上記のようなケースを想定したうえで、評価する必要があると考えます。
従来から知られているWeb脆弱性を検出する手法についてはWebアプリケーション脆弱性診断ガイドラインをご参照ください。
* [Webアプリケーション脆弱性診断ガイドライン](https://github.com/OWASP/www-chapter-japan/tree/master/skillmap_project#web%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E8%84%86%E5%BC%B1%E6%80%A7%E8%A8%BA%E6%96%AD%E3%82%AC%E3%82%A4%E3%83%89%E3%83%A9%E3%82%A4%E3%83%B3)

## 事例紹介

米金融大手Capital Oneにおいて、本記事にて説明しているSSRFの脆弱性を悪用されてしまった事例です。
AWS環境のWAF側の設定ミスにより、SSRFの脆弱性を攻撃によってAWS環境の認証情報を取得されて、S3バケットに格納されている個人情報が漏洩しました。
https://piyolog.hatenadiary.jp/entry/2019/08/06/062154

## 対策

本問題の根本的な原因となるWebアプリケーションの脆弱性については、従来通り実装・設定などによるセキュリティ対策が必要となります。個々の既知のWebアプリケーションの脆弱性に対する対策方法の詳細については、本記事での説明は割愛します。

また、たとえばAWSにおいては以下のような設定変更を行うことで、根本的な対策ではありませんがSSRFの脆弱性に対する緩和策となることが知られています。

* IMDSv2 の使用  
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html

根本的な原因となる脆弱性への対策はもちろん必要となりますが、クラウド環境側から提供されているこのような緩和策の利用も併せて検討すべきでしょう。また、何らかの要因にて認証情報が漏洩することに備え、認証情報の不正利用に対して監視することも重要であると考えます。

## 学習方法/参考文献

* [SSRF(Server Side Request Forgery)徹底入門](https://blog.tokumaru.org/2018/12/introduction-to-ssrf-server-side-request-forgery.html)
* [EC2上でDNS RebindingによるSSRF攻撃可能性を検証した](https://blog.tokumaru.org/2019/03/ssrf-with-dns-rebinding.html)
* [SSRF対策としてAmazonから発表されたIMDSv2の効果と限界](https://blog.tokumaru.org/2019/12/defense-ssrf-amazon-ec2-imdsv2.html)  
本記事にて説明したSSRFについて原理や対策方法について解説された徳丸浩さんのブログ記事です。また、本記事にて取り扱ったSSRFによるAWSの認証情報の取得についても解説されています。

* [SSRF脆弱性を利用したGCE/GKEインスタンスへの攻撃例](https://blog.ssrf.in/post/example-of-attack-on-gce-and-gke-instance-using-ssrf-vulnerability/)  
GCP環境におけるSSRFについて解説されているブログ記事です。

* [Exploiting SSRF in AWS Elastic Beanstalk](https://notsosecure.com/exploiting-ssrf-aws-elastic-beanstalk)  
AWS Elastic Beanstalk・AWS CodePipelineを利用している環境で、SSRFをトリガとして最終的にはRCEまで行うことができたNotSoSecure社の事例です。

* [PenTesterが知っている危ないAWS環境の共通点](https://www.slideshare.net/zaki4649/pentesteraws)  
AWS環境において漏洩した認証情報を利用した攻撃とその対策についてまとめているスライドです。

* [flaws.cloud](http://flaws.cloud/)
* [flaws2.cloud](http://flaws2.cloud/)  
Scott Piper氏(@0xdabbad00)が常設公開されているCTF形式でAWS固有のセキュリティトピックに関して学べるサイトです。

* [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat)
* [AWSGoat](https://github.com/ine-labs/AWSGoat)
* [GCPGoat](https://github.com/ine-labs/GCPGoat)
* [AzureGoat](https://github.com/ine-labs/AzureGoat)  
クラウド環境にまつわる脆弱性を学習するために、脆弱なクラウド環境を構築するためのコンテンツが公開されています。