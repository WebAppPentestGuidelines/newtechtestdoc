---
title: FaaSにおける設定不備と脆弱性の悪用
weight: 999
---

# FaaSにおける設定不備と脆弱性の悪用
## 概要
本稿では、Webやモバイルのアプリケーションで利用されるプログラムを実行するサービスである、FaaSの活用と、実装や利用方法が起因となるアプリケーションの脆弱性について解説します。

## FaaSとは
FaaS(Function as a Service)は、サーバレスなアプリケーションやマイクロサービスで用いられるサービスです。
一定の制約下でプログラムを実行可能にする、IaaSやPaaSのようにクラウド上で利用可能なコンピュータリソースを提供するサービス形態です。
<!-- textlint-disable ja-technical-writing/ja-no-weak-phrase -->
FaaSは多くのクラウドベンダーで提供されており、私たちは気付かないうちにこれらサービスと向き合っているかもしれません。

- [AWS Lambda](https://aws.amazon.com/jp/lambda/)
- [Google Cloud Functions](https://cloud.google.com/functions?hl=ja)
- [Azure Functions](https://azure.microsoft.com/ja-jp/products/functions/)
- [Cloudflare Workers](https://www.cloudflare.com/ja-jp/products/workers/)
- [Netlify Functions](https://www.netlify.com/products/functions/)

FaaSの特徴として、あるイベントをトリガとしてLambdaにホストされたプログラムを実行することが挙げられます。
Amazon Web Services Japan [形で考えるサーバーレス設計](https://aws.amazon.com/jp/serverless/patterns/serverless-pattern/)から例を抜粋して紹介します。

**API Gateway + Lambda + DynamoDBを用いたAPI**<br>
APIをサーバレスで構築する際の基本パターンで、API Gatewayへのリクエストをトリガとして実行されます。
![動的 Web / モバイルバックエンドの図](../image/Pattern-DynamicWeb.3ba6461f647c5156223f5b6710f151c7c542a67a.png)

**S3へアップロード時に画像を加工するためのLambda**<br>
S3 Bucketへ画像やファイルをアップロードしたイベントをトリガとしてLambdaを実行するデザインパターンです。
![画像処理 / シンプルなデータ加工の図](../image/Pattern-S3-processing.0a24e9465dec531156f56ce1c961d00a8e529f3a.png)

**SNS + SQS(Queue) + Lambdaを用いた業務処理**<br>
各種重要処理や競合/重複状態を回避するために利用されるデザインパターンで、SNSのPubSub機能を用いて、SQSに投入し、LambdaがそのイベントをトリガとしてQueueに入ったデータを取得し実行されます。
![イベント駆動の業務処理連携の図](../image/Pattern-Integration.77b207fd045bc2283fe06cb76dc934764ca7114a.png)

# 脆弱性 / 脅威 / 悪用

本稿では、FaaSの利用や設定ミスによって発生するセキュリティ上の脅威や悪用について、下記のページにて触れていきます。また、Injectionやエラー情報の開示といった「アプリケーションの実装」に起因するものは個別には取り扱いませんのでご了承ください。

## 概要

多くのクラウドプロバイダが提供するFaaSでは、プログラムを実行する際に用いる環境変数を設定できる機能が存在します。これら環境変数機能を用いることで、FaaS上で動作するアプリケーション等に値を渡すことが可能です。
この機能では基本的にAPIキーやDBの認証情報、IAMの認証情報などの機密性の高い情報を格納することを非推奨の項目としています。しかし、この環境変数内に認証情報を格納することは禁止されているわけではないため、利用者の実装次第ではこれら情報が含まれている可能性が存在します。
このように環境変数を用いることは、直接的な攻撃につながるわけではありませんが潜在的に認証情報が取得される可能性を高めている状況にあります。

## 原理

FaaSの環境変数に格納された認証情報は、`/proc/self/environ`等から読み取ることが可能で、アプリケーション上の実装の不備をつくことで、入手される可能性があります。

### 擬似環境での解説

例として、AWS Lambda(AWSの提供するFaaS)上で動作する擬似的な環境をもとに解説をします。

**擬似環境の構成図**
![](../image/Pasted-image-20230221233149.png)

このAWS Lambdaには、環境変数に`SECRET_KEY`という環境変数が設定されており、その値には`SECRET_DUMMY`という値が設定されています。そして、動作するコードとして下記のようなJavaScriptのコードが動いています。
このアプリケーションはざっくりな機能として、S3からファイルを取得し、その後Lambda内で加工してクライアントに返すという機能を持っています。
**app.js**
```js
var fs = require('fs');
var AWS = require('aws-sdk');
exports.handler = async (event) => {
    const file = event.queryStringParameters.file;
    try {
        const s3 = new AWS.S3();
        const params = {
            Bucket: 'dummy_bucket_name',
            Key: file
        };
        const s3_file = await s3.getObject(params).promise();
        fs.writeFileSync(`/tmp/${file}`, s3_file.Body);
    }
    catch (err) {
        console.log(err);
    }

    /**
     * ~~画像やファイル操作等の諸々の処理~~
     */

    const env_file_data = fs.readFileSync(`/tmp/${file}`, 'utf8');

    const response = {
        statusCode: 200,
        body: env_file_data
    };
    return response;
};

```
<!-- textlint-disable prh -->
上記のアプリケーションでは、API Gatewayからイベントとして送られてきた、queryStringParametersを直接readFileSyncに利用しており、Path Traversal攻撃が発生する状況にあります。
そのため、`"../../../../proc/self/environ"`と値を入力すると、実装上意図しないローカルファイルが取得できてしまう状況にあります。
<!-- textlint-enable prh -->
そして、実際に攻撃を受けた際は下記のようなレスポンスが返ってきます。

```json
{
  "statusCode": 200,
  "body": "AWS_LAMBDA_FUNCTION_VERSION=$LATEST\u0000AWS_SESSION_TOKEN=...\u0000LD_LIBRARY_PATH=/var/lang/lib:/lib64:/usr/lib64:/var/runtime:/var/runtime/lib:/var/task:/var/task/lib:/opt/lib\u0000AWS_LAMBDA_LOG_GROUP_NAME=/aws/lambda/vulnfunc\u0000LAMBDA_TASK_ROOT=/var/task\u0000AWS_LAMBDA_LOG_STREAM_NAME=2023/02/21/[$LATEST]97a22f53be874c9db31c5ee126df276e\u0000AWS_LAMBDA_RUNTIME_API=127.0.0.1:9001\u0000AWS_EXECUTION_ENV=AWS_Lambda_nodejs14.x\u0000AWS_LAMBDA_FUNCTION_NAME=vulnfunc\u0000AWS_XRAY_DAEMON_ADDRESS=169.254.79.129:2000\u0000PATH=/var/lang/bin:/usr/local/bin:/usr/bin/:/bin:/opt/bin\u0000AWS_DEFAULT_REGION=ap-northeast-1\u0000SECRET_KEY=SECRET_DUMMY\u0000PWD=/var/task\u0000AWS_SECRET_ACCESS_KEY=...\u0000LAMBDA_RUNTIME_DIR=/var/runtime\u0000LANG=en_US.UTF-8\u0000AWS_LAMBDA_INITIALIZATION_TYPE=on-demand\u0000NODE_PATH=/opt/nodejs/node14/node_modules:/opt/nodejs/node_modules:/var/runtime/node_modules:/var/runtime:/var/task\u0000AWS_REGION=ap-northeast-1\u0000TZ=:UTC\u0000AWS_ACCESS_KEY_ID=....\u0000SHLVL=0\u0000_AWS_XRAY_DAEMON_ADDRESS=....\u0000_AWS_XRAY_DAEMON_PORT=2000\u0000_LAMBDA_TELEMETRY_LOG_FD=3\u0000AWS_XRAY_CONTEXT_MISSING=LOG_ERROR\u0000_HANDLER=index.handler\u0000AWS_LAMBDA_FUNCTION_MEMORY_SIZE=128\u0000NODE_EXTRA_CA_CERTS=/etc/pki/tls/certs/ca-bundle.crt\u0000"
}
```

出力された環境変数を整形すると下記のようになります。環境変数内には先に設定した`SECRET_KEY`が含まれており、攻撃者に認証情報を取得されてしまったことがわかります。
```
AWS_LAMBDA_FUNCTION_VERSION=$LATEST
AWS_SESSION_TOKEN=...
LD_LIBRARY_PATH=/var/lang/lib:/lib64:/usr/lib64:/var/runtime:/var/runtime/lib:/var/task:/var/task/lib:/opt/lib
AWS_LAMBDA_LOG_GROUP_NAME=/aws/lambda/vulnfunc
LAMBDA_TASK_ROOT=/var/task
AWS_LAMBDA_LOG_STREAM_NAME=2023/02/21/[$LATEST]97a22f53be874c9db31c5ee126df276e
AWS_LAMBDA_RUNTIME_API=127.0.0.1:9001
AWS_EXECUTION_ENV=AWS_Lambda_nodejs14.x
AWS_LAMBDA_FUNCTION_NAME=vulnfunc
AWS_XRAY_DAEMON_ADDRESS=169.254.79.129:2000
PATH=/var/lang/bin:/usr/local/bin:/usr/bin/:/bin:/opt/bin
AWS_DEFAULT_REGION=ap-northeast-1
SECRET_KEY=SECRET_DUMMY
PWD=/var/task
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=ap-northeast-1
TZ=:UTC
AWS_ACCESS_KEY_ID=....
SHLVL=0
_LAMBDA_TELEMETRY_LOG_FD=3
AWS_XRAY_CONTEXT_MISSING=LOG_ERROR
_HANDLER=index.handler
NODE_EXTRA_CA_CERTS=/etc/pki/tls/certs/ca-bundle.crt
```

## 観点

もしIAMを顧客から受け取っている場合は、FaaSの環境変数に認証情報が含まれていないことを確認してください。
また、下記のいずれかの脆弱性を有している場合、FaaSに設定された認証情報を取得できる場合があるので、認証情報が含まれていないことを確認をしてください。

- Injection攻撃: FaaS上で任意のコマンドやコードを実行可能なOSコマンドやサーバサイドテンプレート等のInjection攻撃が存在する場合、アプリケーションそのものや、構成要素としてFaaSを利用している可能性を考慮して診断してください
- 任意のファイルを読み取ることの可能な攻撃: LFI等のパスに関わる攻撃が存在する場合、アプリケーションそのものや、構成要素としてFaaSを利用している可能性を考慮して診断してください

## 影響

認証情報が漏洩し、攻撃者等に悪用される可能性があります。

## 対策

FaaSで動作するアプリケーションの実装時も、通常のアプリケーション同様の脆弱性対策が必要です。
また、クラウドがFaaSに付与しているIAMの認証情報に関しては、FaaSの特性上、環境変数からアクセスを行うため完全な保護は難しい状況にあります。漏洩した場合のリスクを最小化するために、付与する権限は最小限化したうえで、操作元をそのFaaSのみに設定してください。

## 学習方法/参考文献
- [AWS Lambda 環境変数の使用](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-envvars.html)
- [AWS Lambdaで秘密情報をセキュアに扱う - アンチパターンとTerraformも用いた推奨例の解説](https://blog.flatt.tech/entry/lambda_secret_security)
- [Google Cloud Functionsの環境変数](https://cloud.google.com/functions/docs/configuring/env-var?hl=ja)
- [Cloudflere Workersの環境変数](https://developers.cloudflare.com/workers/platform/environment-variables/)
- [Cloudflere Workersのシークレットの取り扱い](https://developers.cloudflare.com/workers/platform/environment-variables/#add-secrets-to-your-project)
- [OWASP Serverless Top 10](https://owasp.org/www-project-serverless-top-10/)
- [Compromised Cloud Compute Credentials: Case Studies From the Wild](https://unit42.paloaltonetworks.com/compromised-cloud-compute-credentials/)
