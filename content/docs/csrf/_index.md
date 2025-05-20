# CSRF
## 概要
CSRFの対策の1つにSameSite属性が存在します。SameSite属性は、Cookieを配布したサイトとは異なるクロスサイトリクエストへのCookieの送信を制御する属性です。Strict、Lax、Noneの3つが存在し、設定された値により制御が異なります。基本的には、StrictやLaxが設定されている場合、CSRFに対する防御として機能します。しかし、注意するべき点としてSameSite属性は、各ブラウザによりデフォルトの挙動が変わってきます。そのため、ここでは各ブラウザでの挙動の違いを整理していきます。

## 影響
各種ブラウザの挙動におけるSameSite属性の挙動ついて整理していきます。
### Google Chrome
Google Chromeでは2020年2月にChrome80以降でSameSite属性はデフォルトでLaxに変更されています。そのため、デフォルトではクロスサイトに対してCookieは送信されません。また、Noneの場合でもSecure属性が明示的に設定されていない場合は送信されないようになっています。

この仕様は、2024年12月の確認バージョン（131.0.6778.109）でも同様でした。ただし、ChromeやEdgeにおいては、デフォルトの挙動で注意が必要な点があります。それは、Cookieが生成されてから2分間はクロスサイトでCookieが送信される2分間ルールが存在する点です。あります。実際に下記のバージョンのブラウザで確認した所、2分間ルールは顕在しているようです。

| ブラウザ       | バージョン     | 挙動                                                   |
| :------------- | :------------- | :----------------------------------------------------- |
| Google Chrome  | 131.0.6778.109 | 2分間は送信可能、2分間を超えるとCookieは付与されない。 |
| MicroSoft Edge | 1310.2903.112  | 2分間は送信可能、2分間を超えるとCookieは付与されない。 |

また、Chromeでは下記の記事によると2025年現在では、SameSite属性がNoneの場合、サードパーティCookieは送信されるようになっているようです。

・[Next steps for Privacy Sandbox and tracking protections in Chrome](https://privacysandbox.com/news/privacy-sandbox-next-steps/)

### Microsoft Edge
ChromeベースであるMicrosoft Edgeに関してもビルド17672以降のWindows 10で導入されたバージョンはGoogle Chromeと同様の挙動に変更されています。そのため、SameSite属性のデフォルトはLaxに変更されています。また、NoneかつSecure属性が付与されていないとCookieが送信されない挙動もGoogle Chromeと同様となります。これは2024年12月に確認したバージョン（1310.2903.112）でも同様の動作を確認しています。また、こちらでも2分間ルールが存在しています。そして、EdgeでもGoogle Chromeと同様にサードパーティCookieはNoneが設定されている場合は、送信されるようです。

### FireFox
FireFoxは2022年1月11日にGoogle Chromeと同様にSameSite属性のデフォルトの挙動をLaxに変更しました。しかし、FireFox 96.0.3以降においてSameSite属性のデフォルトの挙動は従来の仕様であるNoneに戻されたようです。実際に2024年のバージョン（129.0.1）においてもSameSite属性を指定しない場合にはNoneが設定されています。そのため、デフォルトだとCookieを送信してしまいます。しかし、いくつかのケースではCookieは保護されており実際に影響のあるものは送信されないようになっています。これは、Firefoxで包括的Cookie保護によるブラウザ機能がデフォルトで有効になっているためです。

包括的Cookie保護では、各サイトごとに個別に管理するCookieを分離することで防御を行っています。ただし、ユーザ操作が入るようなformによる操作などの場合はCookieは送信されてしまうので注意が必要となります。また、Firefoxではブラウザのデフォルトの設定によりSameSite属性がNoneでもサードパーティCookieは送信されません。

### Safari
SafariではSameSite属性はデフォルトの設定では「-」と表示され、Noneとなります。しかし、こちらもブラウザの機能によりCookieは送信されません。また、Safariでは全面的にクロスサイトでのサードパーティCookieをブロックしています。本機能をITP（Intelligent Tracking Prevention）と言います。サードパーティCookieを受け取るようにするにはブラウザの設定のプライバシーからWebサイトによるトラッキングを無効にする必要があります。この機能によりSafariではNoneを設定した場合にはCookieが送信されないようになっています。

### 各種ブラウザの挙動のまとめ
各種ブラウザのSameSite属性の挙動

| ブラウザ       | デフォルト | デフォルトの挙動                                             | None時の挙動                                         | サードパーティCookieに関する扱い | 備考                                                         |
| :------------- | :--------- | :----------------------------------------------------------- | :--------------------------------------------------- | :------------------------------- | :----------------------------------------------------------- |
| Google Chrome  | Lax        | LaxなのでCookieは送信されない(条件あり)                      | Secure属性付与時にCookieは送信される                 | Noneの場合、デフォルトは送信     | デフォルトの挙動だと2分間ルールが存在する。                  |
| MicroSoft Edge | Lax        | LaxなのでCookie送信されない(条件あり)                        | Secure属性付与時にCookieは送信される                 | Noneの場合、デフォルトは送信     | デフォルトの挙動だと2分間ルールが存在する。                  |
| FireFox        | None       | NoneなのでCookieは送信されるが、ブラウザの機能により一部制限されている | ブラウザの機能によりCookieの送信は一部制限されている | Noneでもデフォルトは送信されない | 包括的Cookie保護により防御されているのでいくつかのケースではNoneであっても送信されない。 |
| Safari         | None       | Noneだが、ブラウザの機能によりCookieは送信されない           | ブラウザの機能によりCookieは送信されない             | Noneでもデフォルトは送信されない | Webサイトによるトラッキングを有効にしないと送信されない。    |

## 診断観点
現在のブラウザではSameSite属性がLaxに設定されていたり、Noneになっていてもブラウザの別の機能によりある程度は守られているケースがあることがわかりました。しかし、SameSite属性はブラウザの意向やユーザの設定変更により動作が変わります。そのため、SameSite属性による設定を事前に開発側が行なっておく必要があります。具体的には下記の観点で確認が必要です。

1. 明示的にSameSite属性を指定しており、LaxやStrictを付与しているか？
2. GETリクエストを状態更新に関わるAPIで利用していないか？

1は、SameSite属性を指定しない場合、ユーザが利用するブラウザによりデフォルトの挙動に依存します。ユーザのブラウザの種類やバージョンや設定をコンテンツ提供者側が制御することはできません。そのため、状況によってはCSRFが発生してしまう可能性があります。なので、明示的にSameSite属性でLaxやStrictを設定されているかを確認するのが大切です。また、ChromeやEdgeのデフォルトの設定ではCookieが発行されて2分間はトップレベルのリクエストであればCookieを送信してしまいます。2分間という制約があり現実での悪用は難しいですが、この懸念についても明示的に設定することで排除できます。また、SameSite属性が有効な場合でもサブドメインに脆弱性があるケースや中間者攻撃ができる場合には回避されるケースがあるようです。このケースの具体的なケースや対策は参考文献9の記事で詳しく紹介されています。

2は、SameSite属性でLaxを利用している場合に問題となり得ます。LaxではGETリクエストかつトップレベルナビゲーションのリクエストの際にはCookieを送信します。そのため、GETによりデータを更新するような実装がされている場合には、CSRFが発生します。なので、GETリクエストを利用して状態更新をするようなAPIが実装されていないかについて確認する必要があります。

## 対策
デフォルトのブラウザの挙動によりCSRFから守られるケースはあるが明示的に対策を行なっていく必要があるように思います。具体的には下記のような対策を組み合わせて対応してください。

* SameSite属性を明示的にLaxやStrictにする
* データの変更に関わるリクエスト操作ではGETを使わずPOSTを利用する
* リクエストヘッダーを用いた確認を行う（OriginヘッダーやFetch metadataに関するヘッダー）
* CSRFトークンを付与して、チェックする

ヘッダーによる確認としては、OriginヘッダーやFetch metadataがあります。Originヘッダーを用いることでどこから送信されたリクエストであるかを確認できます。これによりリクエストの送信元サイトが同一サイトからのリクエストであるかを確認する手段として用いることができます。また、最近ではさらに詳細なリクエスト元の情報を追加できるFetch metadataが存在しています。Fetch metadataではSec-Fetch-Site、Sec-Fetch-Mode、Sec-Fetch-User、Sec-Fetch-Destの4つが存在しますが、CSRFの対策として利用できるのはSec-Fetch-Siteです。Sec-Fetch-Siteでは、送信元と送信先で同一オリジンなのかクロスサイトなのかについてブラウザ側が設定してくれます。そのため、Sec-Fetch-Siteの値がsame-siteもしくはsame-originであるかをチェックすることで正規のサイトからのリクエストであるかを確認できます。なので、SameSiteの明示的な設定に加えてOriginヘッダーやSec-Fetch-Siteヘッダーに対するチェックを合わせて行うことでCSRFの対策となります。

## 参考文献

1. [新しい SameSite=None; Secure Cookie 設定への対応準備](https://developers.google.com/search/blog/2020/01/get-ready-for-new-samesitenone-secure?hl=ja)

2. [SameSite Update](
   https://www.chromium.org/updates/same-site/)
3. [Cookie とローカル ストレージ](
   https://learn.microsoft.com/ja-jp/microsoftteams/platform/resources/samesite-cookie-update)
4. [2022年1月においてCSRF未対策のサイトはどの条件で被害を受けるか](
   https://blog.tokumaru.org/2022/01/impact-conditions-for-no-CSRF-protection-sites.html)
5. [CookieのSameSite属性と4つの勘違い(2022-10版)](
   https://securesky-plus.com/engineerblog/1809/)
6. [標準モードでの包括的 Cookie 保護の導入](
   https://support.mozilla.org/ja/kb/introducing-total-cookie-protection-standard-mode)
7. [Saying goodbye to third-party cookies in 2024](
   https://developer.mozilla.org/en-US/blog/goodbye-third-party-cookies/)
8. [Full Third-Party Cookie Blocking and More](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)
9.  [SameSite属性とCSRFとHSTS - Cookieの基礎知識からブラウザごとのエッジケースまでおさらいする](https://blog.flatt.tech/entry/samesite_csrf_hsts)

10. [CSRF対策のやり方、そろそろアップデートしませんか](
    https://speakerdeck.com/hiro_y/update-your-knowledge-of-csrf-protection?slide=37)
11. [Cookie の SameSite 属性について](
    https://blog.cybozu.io/entry/2020/05/07/080000)

12. [令和時代の API 実装のベースプラクティスと CSRF 対策](
    https://blog.jxck.io/entries/2024-04-26/csrf.html)

13. [Next steps for Privacy Sandbox and tracking protections in Chrome](
    https://privacysandbox.com/news/privacy-sandbox-next-steps/#:~:text=emerged%2C%20and%20the%20regulatory%20landscape,Chrome%E2%80%99s%20Privacy%20and%20Security%20Settings)

14. [Saying goodbye to third-party cookies in 2024](
    https://developer.mozilla.org/en-US/blog/goodbye-third-party-cookies/)

15. [Strict-Transport-Security](
    https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security)
16. [CSRF、この罠いける？いけない？クイズ](https://www.mbsd.jp/research/20250414/csrf/)

17. [Bypassing SameSite cookie restrictions](
    https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
18.  [Sec-Fetch-Sites](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site)
