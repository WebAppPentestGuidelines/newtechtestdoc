# クラウドストレージサービスにおける設定不備

## 概要

クラウドストレージサービスとは、インターネット上にてファイルの保管・共有などができるサービスです。
画像やJavascriptファイルなどの静的ファイルなどを公開する用途などでも利用されています。
有名処のものとしては以下のようなサービスがあります。

* AWS S3
https://aws.amazon.com/s3/
* Azure Blob Storage
https://azure.microsoft.com/services/storage/blobs/
* Cloud Storage
https://cloud.google.com/storage

### 原因と影響

クラウドストレージサービスにおける設定の不備によって、意図せずインターネット上に情報が公開されてしまうことが問題となっています。
これらのストレージサービスについては、Web上からアクセスするためのURLの形式に規則性が存在しているため、設定不備によって公開されているファイルを探索することが容易となっています。

例えば、設定不備が存在するストレージサービスを探索するための以下のようなツールが公開されています。  
https://github.com/initstring/cloud_enum


### 事例紹介

クラウドストレージサービスの設定不備については、現実世界にて報告されているクラウドサービスに纏わるインシデントの中でも、非常に事例が多い問題です。

* S3の設定ミスにより3TBの空港の機密データが公開されていた事例  
https://www.skyhighsecurity.com/en-us/about/resources/intelligence-digest/unsecured-servers-can-put-lives-at-stake.html

* S3の設定不備に関するHacker Oneのレポート 
https://hackerone.com/reports/128088  
https://hackerone.com/reports/1062803  

* Azure Blob Storageの設定ミスによりMicrosoftの機密データが公開されていた事例  
https://www.vpnmentor.com/blog/report-microsoft-dynamics-leak/

### 対策

### 学習方法/参考文献