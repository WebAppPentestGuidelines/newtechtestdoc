---
title: human readableではないパラメータとの向き合い方（仮）
weight: 999
---

# human readableではないパラメータとの向き合い方（仮）

## はじめに

脆弱性診断では外部から攻撃を受ける可能性のある箇所（アタックサーフェス）(*1)を把握し、内部でどのような処理をしているか知ることが重要です。

Webでは、クエリパラメータやCookie、[フォームデータ](https://developer.mozilla.org/ja/docs/Learn/Forms/Sending_and_retrieving_form_data)などを通じてパラメータをサーバーに送信しますが、　その中には下記のような人間には読みにくい文字列がしばしば登場します。

* `75f96b24-1f28-11ef-b20f-9a438c886500`
* `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Imt1cmVuYWlmIiwiaWF0IjoxNTE2MjM5MDIyfQ.2Y__DLwaJWsCCLJ6F7Cxm9NgXiwIoeQ7Pr7tAePXUT0`
* `6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b`、
* `gASVDAAAAAAAAACMCGt1cmVuYWlmlC4=`

`gASVDAAAAAAAAACMCGt1cmVuYWlmlC4=` は、pythonのpickleを表しているため、外部からこのような文字列を受け取っている場合最悪任意コード実行に繋がる可能性があります。
このように、パラメータを見ただけで脆弱性がある場所を見分けられる可能性があります。

本章ではWebサービスにおいて出現する様々なフォーマットとそれを見た際に何をすればよいのかまとめます

(*1) デジタル庁『政府情報システムにおける脆弱性診断導入ガイドライン』 （2024年1月31日）

## Webシステムにおける様々なエンコーディングとフォーマット

### Base64, 16進数のエンコーディング

Base64エンコーディングは、ASCIIデータに制限されている環境でデータを保存または転送するために使用されます。
Webシステムにおいては、URLのクエリパラメータやCookieに格納する値としてよく利用されています(*2)。

手法としてはデータを64進数に変換し、`A-Za-z0-9+/` や `A-Za-z0-9-_` の63文字に割り当てて実装されています。
また、最後にPaddingとして`=`がつく場合もあります。

Webシステムでは、ASCIIで表現することができないようなバイナリデータを取り扱いたいときにBase64が使われることがあります。
シリアライズしたオブジェクトなどが入っており、[安全ではないでシリアライゼーション](https://portswigger.net/web-security/deserialization)などの脆弱性を悪用するケースや、
デジタル署名がかかっておらず、改ざん可能なデータが入っている可能性があります。

またBase64以外にも16進数エンコーディングをしているケースもあります。
`2F5C9618E721274B32A8`ように、0~9A-Fの16文字を利用します。
こちらもBase64と同様にASCII文字しか使えない環境でエンコーディングしたいときによく用いられる手法です。

こういった文字列が確認できた場合は、4デコードを行い、どのようなデータか確認し、改ざんした場合サーバーにどのような影響を与えられそうか確認することをおすすめします。

(*2) https://datatracker.ietf.org/doc/html/rfc4648

### jwt

JSON Web Token(JWT)とは、JSONオブジェクトとして、情報を安全に送信するためのスタンダードです(*3)。
単にJSONを送るのとは異なり、デジタル署名が入っているため、改ざんすることが難しいトークンです。
WebシステムにおいてはBearerトークンやOIDCのID_TOKEN、ユーザーの状態をJWTとしてCookieに持たせてその情報をもとにサーバーサイドで処理をしたりするような実装がされます。

`xxxxx.yyyyy.zzzzz` のような形式で、`.`で区切られた3つのパートに別れています。
`xxxxx`,`yyyyy`,`zzzzz` はそれぞれBase64エンコードされており、以下のようになります。[(jwt.ioで確認)](https://jwt.io/)
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.KMUFsIDTnFmyG3nMiGM6H9FNFUROf3wh7SmqJp-QV30`

JWTは改ざんが難しいという性質上、クライアントサイドから受け取った情報（権限など）をサーバーサイドで信頼するような実装が多くされます。
2025年現在ではライブラリ側で対策されていることが多く、あまり有効ではありませんが、古いライブラリを利用している場合、署名アルゴリズムとして署名なし(`alg=none`)にする攻撃が挙げられます(*4,*5)。

(3) https://jwt.io/introduction
(4) https://qiita.com/ockeghem/items/cc0b8ab74f46834d055b
(5) https://portswigger.net/web-security/jwt


### UUID

UUIDとは、永続的に一意になるように生成された識別子です(*6)。
`2730488C-D8C0-43CE-A87E-217A61A9667C` のような形式で生成されます。
`-`で囲まれた3つめのフィールドでバージョンが表されており、`43CE`の`4`から、バージョンが4であることがわかります。

Webシステムにおいて、UUIDはユーザーやアップロードしたファイルなどのオブジェクトに割り当てられることが多く、
UUIDの性質から一部の値を改ざんしても特に何も影響がない事が多いです。
もしUUIDを確認したら、[Insecure direct object references(IDOR)](https://portswigger.net/web-security/access-control/idor) などの脆弱性を狙って、
ほかユーザーや権限がないものに対してアクセス権等がないか確認すると脆弱性が見つかるかもしれません。

[6] https://datatracker.ietf.org/doc/html/rfc9562#name-uuid-version-1

### XML

XMLとは、マークアップ言語の一つで、以下のような形式を取ります。(*6)
```
<?xml version="1.0"?>
<!DOCTYPE greeting SYSTEM "hello.dtd">
<greeting>Hello, world!</greeting> 
```

現在でもsvgのような画像ファイルや、データの保存・送信に使われており、
Webにおいては、前述したsvgファイルの取扱の他、POSTリクエストで`<?xml version="1.0" encoding="UTF-8"?><foo>bar</foo>`のようなデータを送信するケースもあります(*7)

XMLはJSON等と違いマークアップ言語であるため、外部エンティティの読み込みなど様々な機能を持っています。
アプリケーションまたはSOAPベースのWebサービスのXMLプロセッサにおいて、ドキュメントタイプ定義（DTD）が有効になっている場合、**Billion Laughs攻撃** や **XXE** といった攻撃に繋がる可能性があります(*8)。

(*6) https://www.w3.org/TR/xml/
(*7) https://portswigger.net/web-security/xxe
(*8) https://github.com/OWASP/Top10/blob/master/2017/ja/0xa4-xxe.md

### 乱数生成
 
#### 安全でない乱数の利用

Webシステムでは、初期パスワードやワンタイムトークンなどで乱数を利用する場合があります。
乱数には暗号論的擬似乱数生成器と呼ばれる乱数と、そうでない乱数があります。
安全でない乱数を利用すると、一定の情報を収集することで、次回以降生成される値を予測することができる可能性があります。

例えば、Javaの場合[Random](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Random.html)は安全でない乱数で、[SecureRandom](https://docs.oracle.com/javase/jp/8/docs/api/java/security/SecureRandom.html) は 暗号用に強化された乱数ジェネレータ であり、予測されては困る値は SecureRandom で生成したほうが良いことがわかります。

phpはメルセンヌ・ツイスタと呼ばれる乱数を利用していますが、初期条件のパターン数が少ないため、[openwallが出しているツール](https://github.com/openwall/php_mt_seed) を利用することで、少ない情報から高速に内部状態を復元し、次回以降生成される値を予測することができる可能性があります。

### ハッシュ化
 
Webシステムでは、一致のみ判断できれば良く、元の値に復元されてほしくない場合、ハッシュ化を利用することがあります。

Webシステムにおいて、ハッシュ化は主にパスワードで利用されています。
パスワードハッシュとしてはbcryptと呼ばれるアルゴリズムがよく用いられています。
その他にも、パスワードリセットなどでユーザーを識別するIDとしてメールアドレスをハッシュ化したものなどを利用したりしており、以下のような文字列が露出しているケースがあります。

md5(128bit): d41d8cd98f00b204e9800998ecf8427e
sha256(256bit): d14a028c2a3a2bc9476102bb288234c415a2b01f828ea62ac5b3e42f

Webサービスからこういったハッシュ値が露出している場合、ユーザーの何かを識別している可能性があるため他のハッシュ値と入れ替えてみてどのような挙動になるのか確認するとIDORなどの脆弱性を発見できる可能性があります。

また、bcryptの仕様として72文字までしかハッシュ化することができず、
多くのライブラリでは73文字以降は切り捨てにななっているため、
prefixが入っているとハッシュとして機能しない場合が考えられます。

### 暗号化

ハッシュ化したデータから元の値に復元することはハッシュ化した本人でも困難ですが、暗号化では復号することができる鍵を持っていれば元のデータを復元することができます。
Webシステムでは、クライアントにCookieなどで状態を持たせたいが、中身はクライアントに知らせたくない場合に利用されることがあります。
こういった実装ではサーバーサイドで暗号化と復号を行うため、暗号化と復号で同じ鍵を利用する共通鍵暗号を利用することがよくあります。

共通鍵暗号を利用している場合、鍵の漏洩は平文の漏洩につながるため、診断では特にモバイルアプリなどの場合、鍵がクライアントに露出していないか確認しましょう。
よく用いられるAESでは、128bit、192ビット、256ビットの3種類の鍵があります。
これらの長さのバイト列で暗号化に関連してそうな場合、鍵の可能性があるため、復号できないか確認してみるのも良いかもしれません。

共通鍵暗号でよく用いられる暗号利用モード毎にそれぞれ特徴と診断時のポイントを以下に示します。

#### AES-CBC

laravelなどで利用されている暗号利用モードで、暗号化というとこの暗号利用モードになるフレームワークも少なくありません。
しかしフレームワークによって微妙に扱いが異なり、似たようなソースコードでも脆弱になったり脆弱にならなかったりと意外と扱いが難しい暗号利用モードです。
フレームワークを利用して暗号化/復号を実装していても脆弱になるケースがあるので注意して運用しましょう。

例を上げるとこちらのコードは復号に失敗、JSONのDecodeに失敗していることを通知しているだけですが、Padding Oracle Attackに対して脆弱です。(https://github.com/kurenaif/padding_oracle_training/blob/main/problem/app/app.rb#L64-L83)

特徴としては以下が挙げられます。

 * 暗号文の長さは16bytesの倍数になる
 * 暗号文に加えてivも付与されていることが多く、セットで16bytes増えたものを取り扱うことが多い
 * ivが固定だとAES-ECBと似たような攻撃に対して脆弱になる可能性がある
 * 平文に対してに必ずPaddingが付与されており、復号時にそれを除去する必要がある
 * Paddingを除去する際に特定のフォーマットに従っていないことをユーザに知られると、Padding Oracle Attackの脆弱性に繋がり、平文の漏洩と改ざんが可能になる。
 * これ単体で改ざんを検知する仕組みが無く、特に最初の16bytesはivを変更することで容易に変更可能である。

https://rintaro.hateblo.jp/entry/2017/12/31/174327
https://github.com/kurenaif/ctf_lesson/blob/master/cbc/cbc_padding_oracle/server.py
https://github.com/kurenaif/ctf_lesson/blob/master/cbc/cbc.pdf
https://github.com/kurenaif/padding_oracle_training

#### AES-ECB

ivを使わなくて良い、昔からあるライブラリ等の理由で、時々見ます。
安全面を考えると基本的に使ってほしくないが、Padding Oracle AttackができるAES-CBCよりは攻撃面でできることが少ないです。

特徴としては以下が挙げられます。

 * 暗号文の長さは16bytesの倍数になる
 * ivは不要
 * 平文に対してに必ずPaddingが付与されており、復号時にそれを除去する必要がある
 * 同じ平文のブロックがあったときに同じ暗号文になる
 * 暗号文をうまく入れ替えることで該当箇所の平文を入れ替えることができる

https://ja.wikipedia.org/wiki/%E6%9A%97%E5%8F%B7%E5%88%A9%E7%94%A8%E3%83%A2%E3%83%BC%E3%83%89
https://github.com/kurenaif/ctf_lesson/blob/master/cbc/cbc.pdf

#### AES-GCM

デフォルトでGCMになるライブラリは見ない印象ですが、AES-CBCでの問題点を解決しようとしたときに候補に上がる暗号利用モードです。
この暗号利用モードを利用することで、改ざん検知を同時にすることができます。

特徴としては以下が挙げられます。

 * 暗号文は16bytesの倍数でなくても良い。
 * ivを使いまわしている場合は鍵無しで暗号文を平文に戻したり、改ざんをすることができる
 * 改ざん検知に利用されるmacのバイト数が少ない場合、macの前方一致で検証されるため不十分な検証になる実装がよくある
 * 暗号文と一緒にivとmacをセットで取り扱うことが多い

https://jovi0608.hatenablog.com/entry/20160524/1464054882
https://github.com/SECCON/SECCON2022_online_CTF/tree/main/crypto/witches_symmetric_exam
https://github.com/kurenaif/ctf_lesson/tree/master/aes-gcm

### AES-ECBを用いることにより生じる脆弱性とその攻撃方法

#### 同じブロックの平文は同じ暗号文になる

この問題はECBを利用している限り避けられません。
16bytesごとに区切り暗号化をするため、同じ暗号文であれば同じ平文になります。
この特性を利用して同じ16bytesの暗号文が繰り返し出ている場合、同じ平文があることがわかります。

また下図のように攻撃者の入力と、知られてほしくない文字列が連携されている場合、1文字ずつずらしながらブルートフォースをすることで、暗号化された文字を漏洩させることも可能です。

![image](https://hackmd.io/_uploads/S1jcIXPV0.png)



#### （完全性の損害）ブロックと入れ替えることができる。

ECBでは、ブロック同士が互いに干渉しないため、ブロック同士の入れ替えが可能になります。
下図では、`Admin:False` となっているものを、攻撃者の入力したブロックと入れ替えることにより`Admin:True`に変更する例を取り上げています。

![image](https://hackmd.io/_uploads/ryiQDmw4A.png)


### AES-CBCの概要とその特徴
    
    
#### （完全性の損害）ivをbitflipすることで先頭16bytesを改ざんすることができる

AES-CBCでは、ivを変更することで先頭1ブロックを容易に改ざんすることが可能です。
図に示す1ブロックだけの例を見てみると、平文の前にivがxorされているため、このivを変えることにより復号される平文を好きに変更できることがわかります。

![image](https://hackmd.io/_uploads/rk5ntmv40.png)


平文が長くなった場合でも先頭16bytesは同様に変更することができるため、ここに改ざんされると致命的な情報を入れて運用している場合は大きな影響が出る可能性があります。

#### （機密性・完全性の損害）復号したPaddingが正しいかどうかを攻撃者が知ることができる場合、暗号文の改ざんや復号ができる可能性がある

AES-CBCを復号し平文を得る過程は、復号→Paddingの除去という流れになっています。
ここで、Paddingのフォーマットが守られていないとライブラリ等でエラーが生じるわけですが、
「ここでエラーが生じたこと」がユーザにバレてしまうと平文の漏洩や攻撃者の臨んだバイト列に改ざんすることが可能になります。

例として、最後から2つめのブロックの最後1バイト（赤色で示した部分）を変更する攻撃を紹介します。
Paddingのフォーマット上、最後の1バイトが\x01で終わっている場合、Paddingのチェックは通り、復号は成功します。しかしながら、改ざん後でjsonデコード等を行っている場合、最後から2ブロック目がランダムなバイト列になるため、エラーが発生する可能性があります。
ここで、復号時のエラー(Padding)のエラーなのか、jsonデコード等のエラーなのかを攻撃者が知ることができる場合、Padding Oracle Attackが可能になります。(https://github.com/kurenaif/padding_oracle_training/blob/main/problem/app/app.rb#L64-L83)

Padding Oracle Attackが成功すると、平文の漏洩、及び改ざん可能になります。

![image](https://hackmd.io/_uploads/rJPysmw4R.png)


### AES-GCMの概要とその特徴

AES-ECBやCBCのような形式ではどうしても「それ単体では改ざんに耐性がない」という弱点が出てきてしまいます。
このような攻撃手法が存在しているため、暗号文をユーザーがいじれる場合macはほぼ必須なのですが、暗号化をしていると完全性も保たれると思い実装してしまう気持ちもわかります。
そこで生まれたのがAEADで、そのうちの一つがGCMです。
AES-GCMでは暗号化・復号と改ざん検知を同時に行うことができます。

#### （機密性・完全性の損害）ivを使いまわしている場合、データ改ざんすることができ、更に平文も取得できる

GCMでivを使いまわしている場合、データを改ざんすることができ、更に平文も取得することができます。
ほとんどのライブラリではivは自動的に生成されるようになっており、ivの使い回しは起きにくいものと考えています。

https://jovi0608.hatenablog.com/entry/20160524/1464054882