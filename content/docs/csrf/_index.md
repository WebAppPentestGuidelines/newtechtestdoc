# CSRF
## 概要
CSRFの対策の1つにSameSite属性が存在します。SameSite属性は、Cookieを配布したサイトとは異なるクロスサイトリクエストへのCookieの送信を制御する属性です。Strict/Lax/Noneの3つが存在し、設定された値により制御が異なります。基本的には、StrictやLaxが設定されている場合、CSRFに対する防御として機能します。しかし、注意するべき事項としてSameSite属性は、各ブラウザによりデフォルトの挙動が変わってきます。そのため、ここでは各ブラウザでの挙動の違いを整理していきます。

## 影響
各種ブラウザの挙動におけるSameSite属性の挙動ついて整理していきます。
### Google Chrome
Google Chromeでは2020年2月にChrome80以降でSameSite属性はデフォルトでLaxに変更されています。
そのため、デフォルトではクロスサイトに対してCookieは送信されません。また、Noneの場合でもSecure属性が明示的に設定されていない場合は送信されないようになっています。

>Chromeに関する動作
>"新しい SameSite=None; Secure Cookie 設定への対応準備"、Google検索セントラル
>https://developers.google.com/search/blog/2020/01/get-ready-for-new-samesitenone-secure?hl=ja
>"SameSite Update"、The Chromium Projects
>https://www.chromium.org/updates/same-site/

この仕様は、2024年12月に確認したバージョン(131.0.6778.109)でも同様です。ただし、ChromeやEdgeにおいては、デフォルトの設定では注意が必要な点があります。それは、Cookieが生成されてから2分間はクロスサイトでCookieが送信される2分間ルールがあることです。2分間の間はクロスサイトでCookieを送信することができます。実際に下記のバージョンのブラウザで確認した所、現在も2分間ルールは顕在しているようです。

| ブラウザ       | バージョン     | 挙動                                                   |
| :------------- | :------------- | :----------------------------------------------------- |
| Google Chrome  | 131.0.6778.109 | 2分間は送信可能、2分間を超えるとCookieは付与されない。 |
| MicroSoft Edge | 1310.2903.112  | 2分間は送信可能、2分間を超えるとCookieは付与されない。 |

また、Chromeでは下記のブログから2025年現在では、SameSite属性がNoneの場合、サードパーティCookieは送信されるようになっているようです。

> Anthony Chavez、Next steps for Privacy Sandbox and tracking protections in Chrome、https://privacysandbox.com/news/privacy-sandbox-next-steps/#:~:text=emerged%2C%20and%20the%20regulatory%20landscape,Chrome%E2%80%99s%20Privacy%20and%20Security%20Settings

### Microsoft Edge
ChromeベースであるMicrosoft Edgeに関してもビルド17672以降のWindows 10で導入されたバージョンはGoogle Chromeと同様の挙動に変更されています。そのため、SameSite属性のデフォルトはLaxに変更されています。
また、NoneかつSecure属性が付与されていないとCookieが送信されない挙動もGoogle Chromeと同様です。
これは2024年12月に確認したバージョン(1310.2903.112)でも確認されています。その他にもこちらでも2分間ルールが存在しています。

>Microsoft　Edgeに関する動作
>"Cookie とローカル ストレージ"、Microsoft Learn
https://learn.microsoft.com/ja-jp/microsoftteams/platform/resources/samesite-cookie-update

また、EdgeでもGoogle Chromeと同様にサードパーティCookieはNoneが設定されている場合は、送信されます。

### FireFox
FireFoxは2022年1月11日にGoogle Chromeと同様にSameSite属性のデフォルトの挙動をLaxに変更しました。
しかし、FireFox 96.0.3以降においてSameSite属性のデフォルトの挙動は従来の仕様であるNoneに戻されたようです。

> FireFoxの挙動に関する確認資料
> 徳丸浩、"2022年1月においてCSRF未対策のサイトはどの条件で被害を受けるか"、徳丸浩の日記
https://blog.tokumaru.org/2022/01/impact-conditions-for-no-CSRF-protection-sites.html
>坂本、"CookieのSameSite属性と4つの勘違い(2022-10版)"、セキュアスカイプラス
https://securesky-plus.com/engineerblog/1809/

実際に2024年のバージョン(129.0.1)においてもSameSite属性を指定しない場合にはNoneが設定されています。そのため、デフォルトだとCookieを送信してしまいます。しかし、いくつかのケースではCookieは保護されており実際に影響のあるものは送信されないようになっています。これは、Firefoxで包括的Cookie保護によるブラウザ機能がデフォルトで有効になっているためです。

> 包括的Cookie保護に関する資料
> "標準モードでの包括的 Cookie 保護の導入"、Support moz://a
> https://support.mozilla.org/ja/kb/introducing-total-cookie-protection-standard-mode

包括的Cookie保護では、各サイトごとに個別に管理するCookieを分離することで防御を行っています。ただし、ユーザ操作が入るようなformによる操作などの場合はCookieは送信されてしまうので注意が必要となります。また、Firefoxではブラウザのデフォルトの設定によりSameSite属性がNoneでもサードパーティCookieは送信されません。

> Chris Mills、Saying goodbye to third-party cookies in 2024
> https://developer.mozilla.org/en-US/blog/goodbye-third-party-cookies/

### Safari
SafariではSameSite属性はデフォルトの設定では「-」と表示され、Noneとなります。しかし、こちらもブラウザの機能によりCookieは送信されません。また、Safariでは全面的にクロスサイトでのサードパーティCookieをブロックしています。本機能をITP（Intelligent Tracking Prevention）と言います。サードパーティCookieを受け取るようにするにはブラウザの設定のプライバシーからWebサイトによるトラッキングを無効にする必要があります。この機能によりSafariではNoneを設定した場合にはCookieが送信されないようになっています。

> Safariの挙動に関する確認資料
>John Wilander、"Full Third-Party Cookie Blocking and More"、WebKit
https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/

### 各種ブラウザの挙動のまとめ
各種ブラウザのSameSite属性の挙動

| ブラウザ       | デフォルト | デフォルトの挙動                                             | None時の挙動                                         | サードパーティCookieに関する扱い | 備考                                                         |
| :------------- | :--------- | :----------------------------------------------------------- | :--------------------------------------------------- | :------------------------------- | :----------------------------------------------------------- |
| Google Chrome  | Lax        | LaxなのでCookieは送信されない(条件あり)                      | Secure属性付与時にCookieは送信される                 | Noneの場合、デフォルトは送信     | デフォルトの挙動だと2分間ルールが存在する。                  |
| MicroSoft Edge | Lax        | LaxなのでCookie送信されない(条件あり)                        | Secure属性付与時にCookieは送信される                 | Noneの場合、デフォルトは送信     | デフォルトの挙動だと2分間ルールが存在する。                  |
| FireFox        | None       | NoneなのでCookieは送信されるが、ブラウザの機能により一部制限されている | ブラウザの機能によりCookieの送信は一部制限されている | Noneでもデフォルトは送信されない | 包括的Cookie保護により防御されているのでいくつかのケースではNoneであっても送信されない。 |
| Safari         | None       | Noneだが、ブラウザの機能によりCookieは送信されない           | ブラウザの機能によりCookieは送信されない             | Noneでもデフォルトは送信されない | Webサイトによるトラッキングを有効にしないと送信されない。    |

## 診断観点
現在のブラウザではSameSite属性がLaxに設定されていたり、Noneになっていてもブラウザの別の機能によりある程度は守られているケースがあることがわかりました。しかし、SameSite属性はブラウザの意向やユーザの設定変更により挙動が変化することになります。そのため、SameSite属性による設定を事前に開発側が行なっておく必要があります。具体的には下記の観点で確認が必要です。

1. 明示的にSameSite属性を指定しており、LaxやStrictを付与しているか？
2. GETリクエストを状態更新APIで利用していないか？

1は、SameSite属性を指定しない場合、ユーザが利用するブラウザによりデフォルトの挙動が変わってきます。ユーザ側のブラウザの種類やバージョンや設定をコンテンツ提供者側が制御することはできません。そのため、状況によってはCSRFが発生してしまう可能性があります。なので、明示的にSameSite属性でLaxやStrictを設定することが必要です。また、ChromeやEdgeのデフォルトの設定ではCookieが発行されて2分間はトップレベルのリクエストであればCookieを送信してしまいます。2分間という制約があり現実での悪用は難しいですが、この懸念についても排除できます。ただ、注意するべき事項としてSameSite属性が有効な場合でもサブドメインに脆弱性があるケースや中間者攻撃ができる場合には回避されるケースがあります。具体的なケースはGMO Flatt Securityの@okazu_dm さんが記載しているブログが参照してみてください。本ブログで示されているようにHSTS(HTTP Strict Transport Security)を有効にして、HTTPSによる有効化により回避されないようにすることを提案しています。

> SameSite属性とCSRFとHSTS - Cookieの基礎知識からブラウザごとのエッジケースまでおさらいする
> https://blog.flatt.tech/entry/samesite_csrf_hsts

2は、SameSite属性でLaxを利用している場合に問題となり得ます。LaxではGETリクエストかつトップレベルナビゲーションのリクエストの際にはCookieを送信します。そのため、GETによりデータを更新するような実装がされている場合には、CSRFが発生します。なので、GETリクエストを利用して状態更新をするようなAPIが実装されていないかについて確認する必要があります。


## SameSite属性による各種挙動における攻撃事例を示したサイト
小山凌弥、CSRF、この罠いける？いけない？クイズ
https://www.mbsd.jp/research/20250414/csrf/

Bypassing SameSite cookie restrictions、PortSwigger
https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions

## 対策
デフォルトのブラウザの挙動によりCSRFから守られるケースはあるが明示的に対策を行なっていく必要があるように思います。具体的には下記のような対策をしてください。

* SameSite属性を明示的にLaxやStrictにする
* データの変更に関わるリクエスト操作ではGETを使わずPOSTを利用する
* リクエストヘッダーを用いた確認を行う。(OriginヘッダーやFetch metadata)
* CSRFトークンを付与して、チェックする。

ヘッダーによる確認としては、OriginヘッダーやFetch metadataがある。Originヘッダーはどこからきたリクエストであるかを確認することができます。これによりリクエストサイトが同一サイトからのリクエストであるかを確認する手段として用いることができます。また、最近ではよりリクエスト元の情報を追加できるFetch metadataが存在しています。Fetch metadataではSec-Fetch-Site、Sec-Fetch-Mode、Sec-Fetch-User、Sec-Fetch-Destの4つが存在しますが、CSRFの対策として利用できるのはSec-Fetch-Siteです。Sec-Fetch-Siteでは、送信元と送信先で同一オリジンなのかクロスサイトなのかについてブラウザ側が設定してくれます。

Sec-Fetch-Sites
https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site


そのため、Sec-Fetch-Siteの値がsame-siteもしくはsame-originであるかをチェックすることで正規のサイトからのリクエストであるかを確認できます。なので、SameSiteの設定に加えてoriginやSec-Fetch-Siteヘッダーに対するチェックを合わせて行うことはCSRFの対策となります。また、加えてHSTSヘッダーによりHTTPSを強制したり、トークンによる防御機構を合わせて持つとよりカバーされる領域が増えていきます。セキュリティではデフォルトの挙動にだけ依存せず、複数の防御機構を加えて対策を行うことが大切です。


## 参考

* YAMAOKA Hiroyuki、CSRF対策のやり方、そろそろアップデートしませんか、https://speakerdeck.com/hiro_y/update-your-knowledge-of-csrf-protection?slide=37
* 小林、Cookie の SameSite 属性について、https://blog.cybozu.io/entry/2020/05/07/080000
* jxck、令和時代の API 実装のベースプラクティスと CSRF 対策、https://blog.jxck.io/entries/2024-04-26/csrf.html
* Anthony Chavez、Next steps for Privacy Sandbox and tracking protections in Chrome、https://privacysandbox.com/news/privacy-sandbox-next-steps/#:~:text=emerged%2C%20and%20the%20regulatory%20landscape,Chrome%E2%80%99s%20Privacy%20and%20Security%20Settings
* Chris Mills、Saying goodbye to third-party cookies in 2024、https://developer.mozilla.org/en-US/blog/goodbye-third-party-cookies/
* Strict-Transport-Security、https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security
