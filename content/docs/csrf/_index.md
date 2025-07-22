---
title: CSRF
weight: 999

---
# CSRF

## 概要
CSRFの対策の1つにSameSite属性が存在します。SameSite属性は、Cookieを配布したサイトとは異なるクロスサイトリクエストへのCookieの送信を制御する属性です。Strict、Lax、Noneの3つが存在し、設定された値により制御が異なります。基本的には、StrictやLaxが設定されている場合、CSRFに対する防御として機能します。しかし、注意するべき点としてSameSite属性は、各ブラウザによりデフォルトの挙動が変わってきます。そのため、ここでは各ブラウザにおけるSameSite属性の挙動について整理していきます。

## SameSite属性の挙動比較
### Google Chrome
Google Chromeでは2020年2月にGoogle Chrome80以降でSameSite属性はデフォルトでLaxに変更されています。（※1）そのため、デフォルトではクロスサイトに対してCookieは送信されません。また、Noneの場合でもSecure属性が明示的に設定されていない場合は送信されないようになっています。この仕様は、2024年12月バージョン（131.0.6778.109）でも同様の挙動であることを確認しています。

また、2025年現在Google ChromeではSameSite属性がNoneの場合、サードパーティCookieは送信されるようです。（※3）

### Microsoft Edge
Google ChromeベースであるMicrosoft Edgeに関してもビルド17672以降のWindows 10で導入されたバージョンはGoogle Chromeと同様の挙動に変更されています。（※4）そのため、SameSite属性のデフォルトはLaxに変更されています。また、NoneかつSecure属性が付与されていないとCookieが送信されない挙動もGoogle Chromeと同様です。これは2024年12月のバージョン（1310.2903.112）でも同様の挙動であることを確認しています。

また、Microsoft EdgeでもGoogle Chromeと同様にサードパーティCookieはNoneが設定されている場合は、送信されるようです。

### [コラム]Google ChromeとMicrosoft Edgeで発生する2分間ルールについて

Google ChromeとMicrosoft Edgeには、SameSite属性が未指定の場合に限り、発行から2分間はクロスサイトリクエストでもCookieを送信する挙動が存在します。この挙動は、2分間ルールと呼ばれます。実際に下記のブラウザのバージョンで確認した所、2分間ルールは現在も発生することが確認できました。

| ブラウザ       | バージョン     | 挙動                                                   |
| :------------- | :------------- | :----------------------------------------------------- |
| Google Chrome  | 131.0.6778.109 | 2分間は送信可能、2分間を超えるとCookieは付与されない。 |
| Microsoft Edge | 1310.2903.112  | 2分間は送信可能、2分間を超えるとCookieは付与されない。 |

### Firefox
Firefoxは2022年1月11日にGoogle Chromeと同様にSameSite属性のデフォルトの挙動をLaxに変更しました。しかし、Firefox 96.0.3以降においてSameSite属性のデフォルトの挙動は従来の仕様であるNoneに戻されたようです。（※5、※6）

実際に2024年のバージョン（129.0.1）においてもSameSite属性を指定しない場合にはNoneが設定されており、クロスサイトリクエストでもCookieを送信する可能性があります。しかし、いくつかのケースではCookieは保護されており実際に影響のあるものは送信されないようになっています。これは、Firefoxで包括的Cookie保護によるブラウザ機能がデフォルトで有効になっているためです。（※7）包括的Cookie保護では、各サイトごとに個別に管理するCookieを分離することで防御しています。ただし、formによる操作などユーザによる操作が発生する場合は、Cookieが送信されてしまうので注意してください。

また、Firefoxではブラウザのデフォルトの設定によりSameSite属性がNoneでもサードパーティCookieは送信されません。（※8）

### Safari
SafariではSameSite属性はデフォルトの設定ではNone相当で扱われます。そのため、Firefoxと同様にブラウザの機能によりCookieは送信されません。ただし、Safariの開発者ツール上では「None」ではなく、「-」と表示されます。

また、Safariでは全面的にクロスサイトでのサードパーティCookieをブロックしています。本機能をITP（Intelligent Tracking Prevention）と言います。サードパーティCookieを受け取るようにするにはSafariのプライバシー設定でWebサイトによるトラッキングを無効にする必要があります。この機能により、SameSite属性が未設定、またはNoneを設定した場合にはCookieが送信されないようになっています。（※9）

### 各ブラウザにおける挙動のまとめ
各ブラウザにおけるSameSite属性やサードパーティCookieの扱いに関しては下記の表のようになります。

| ブラウザ       | デフォルト | デフォルトの挙動                                             | None時の挙動                                         | サードパーティCookieに関する扱い | 備考                                                         |
| :------------- | :--------- | :----------------------------------------------------------- | :--------------------------------------------------- | :------------------------------- | :----------------------------------------------------------- |
| Google Chrome  | Lax        | LaxなのでCookieは送信されない(条件あり)                      | Secure属性付与時にCookieは送信される                 | Noneの場合、デフォルトは送信     | デフォルトの挙動だと2分間ルールが存在する。                  |
| Microsoft Edge | Lax        | LaxなのでCookie送信されない(条件あり)                        | Secure属性付与時にCookieは送信される                 | Noneの場合、デフォルトは送信     | デフォルトの挙動だと2分間ルールが存在する。                  |
| Firefox        | None       | NoneなのでCookieは送信されるが、ブラウザの機能により一部制限されている | ブラウザの機能によりCookieの送信は一部制限されている | Noneでもデフォルトは送信されない | 包括的Cookie保護により防御されているのでいくつかのケースではNoneであっても送信されない。 |
| Safari         | None       | Noneだが、ブラウザの機能によりCookieは送信されない           | ブラウザの機能によりCookieは送信されない             | Noneでもデフォルトは送信されない | Webサイトによるトラッキングを有効にしないと送信されない。    |

## 診断観点
現在のブラウザではSameSite属性がLaxに設定されていたり、Noneになっていてもブラウザの別の機能によりある程度は守られることがわかりました。しかし、SameSite属性はブラウザの仕様変更やユーザの設定により動作が変わります。そのため、SameSite属性による設定を事前に開発側が行っておく必要があります。具体的には下記の観点で確認が必要です。

1. 明示的にSameSite属性を指定しており、LaxやStrictを付与しているか
2. GETリクエストを状態更新に関わるAPIで利用していないか

1を確認する理由は、SameSite属性を指定しない場合、その挙動は利用者のブラウザの種類やバージョン、設定に依存し、コンテンツ提供側で制御できないためです。その結果、状況によってはCSRFが発生してしまいます。ですので、明示的にSameSite属性でLaxやStrictを設定されているかを確認するのが大切です。特にGoogle ChromeやMicrosoft Edgeでは、明示的にSameSite属性を指定しないと発行されてから2分間はCookieが送信される点に注意してください。2分間という制約があり現実での悪用は難しいですが、この懸念についても明示的に設定することで排除できます。また、SameSite属性が有効な場合でもサブドメインに脆弱性があるケースや中間者攻撃ができる場合には回避されるケースがあるようため注意してください。（※13）

2は、SameSite属性でLaxを利用している場合に問題となり得ます。LaxではGETリクエストかつトップレベルナビゲーションのリクエストの際にはCookieを送信します。そのため、GETによりデータを更新するような実装がされている場合には、CSRFが発生します。ですので、GETリクエストを利用して状態更新をするようなAPIが実装されていないかについて確認する必要があります。

## 対策
ブラウザのデフォルトの機能によりCSRFから守られるケースはありますが、利用しているブラウザやユーザの設定状況によっては被害が発生する可能性もあります。そのため、アプリケーション側で対策をしておくことが大切です。具体的には下記のような対策を組み合わせて対応する必要があります。

* SameSite属性に明示的にLaxやStrictを設定する
* データの変更に関わるリクエスト操作ではGETを使わずPOSTを利用する
* リクエストヘッダを用いて確認する（OriginヘッダやFetch metadataを用いた確認）
* 従来通りCSRFトークンを付与して、チェックする

ヘッダによる確認としては、OriginヘッダやFetch metadataがあります。Originヘッダを用いることでどこから送信されたリクエストであるかを確認できます。これによりリクエストの送信元サイトが同一サイトからのリクエストであるかを確認する手段として用いることができます。また、最近ではさらに詳細なリクエスト元の情報を追加できるFetch metadataが存在しています。Fetch metadataでは下記の4つがあります。

* Sec-Fetch-Site
* Sec-Fetch-Mode
* Sec-Fetch-User
* Sec-Fetch-Dest

CSRFの対策として利用できるのはSec-Fetch-Siteです。Sec-Fetch-Siteでは、送信元と送信先で同一オリジンなのかクロスサイトなのかについてブラウザ側が設定してくれます。（※14）そのため、Sec-Fetch-Siteの値がsame-siteもしくはsame-originであるかをチェックすることで正規のサイトからのリクエストであるかを確認できます。そのため、SameSiteの明示的な設定に加えてOriginヘッダやSec-Fetch-Siteヘッダに対するチェックを合わせて行うことでCSRFの対策となります。

## 参考文献

1. [新しい SameSite=None; Secure Cookie 設定への対応準備](https://developers.google.com/search/blog/2020/01/get-ready-for-new-samesitenone-secure?hl=ja)

2. [SameSite Update](
   https://www.chromium.org/updates/same-site/)
3. [Next steps for Privacy Sandbox and tracking protections in Google Chrome](https://privacysandbox.com/news/privacy-sandbox-next-steps/)
4. [Cookie とローカル ストレージ](
   https://learn.microsoft.com/ja-jp/microsoftteams/platform/resources/samesite-cookie-update)
5. [2022年1月においてCSRF未対策のサイトはどの条件で被害を受けるか](
   https://blog.tokumaru.org/2022/01/impact-conditions-for-no-CSRF-protection-sites.html)
6. [CookieのSameSite属性と4つの勘違い(2022-10版)](
   https://securesky-plus.com/engineerblog/1809/)
7. [標準モードでの包括的 Cookie 保護の導入](
   https://support.mozilla.org/ja/kb/introducing-total-cookie-protection-standard-mode)
8. [Saying goodbye to third-party cookies in 2024](
   https://developer.mozilla.org/en-US/blog/goodbye-third-party-cookies/)
9. [Full Third-Party Cookie Blocking and More](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)
10. [Cookie の SameSite 属性について](
    https://blog.cybozu.io/entry/2020/05/07/080000)
11. [Bypassing SameSite cookie restrictions](
    https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
12. [Saying goodbye to third-party cookies in 2024](
    https://developer.mozilla.org/en-US/blog/goodbye-third-party-cookies/)
13.  [SameSite属性とCSRFとHSTS - Cookieの基礎知識からブラウザごとのエッジケースまでおさらいする](https://blog.flatt.tech/entry/samesite_csrf_hsts)
14.  [Sec-Fetch-Sites](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site)
15. [CSRF対策のやり方、そろそろアップデートしませんか](
    https://speakerdeck.com/hiro_y/update-your-knowledge-of-csrf-protection)
16. [CSRF、この罠いける？いけない？クイズ](https://www.mbsd.jp/research/20250414/csrf/)
17. [令和時代の API 実装のベースプラクティスと CSRF 対策](
    https://blog.jxck.io/entries/2024-04-26/csrf.html)
