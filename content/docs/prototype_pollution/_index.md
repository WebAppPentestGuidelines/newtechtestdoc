---
title: Prototype Pollution
weight: 999
---
# Prototype Pollution

## 概要

JavaScriptでは主に"継承"を実現するための[`__proto__`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)プロパティが存在します。
Prototype Pollution は、この`__proto__`プロパティを通じて特定のオブジェクトの内容を不正に変更する攻撃手法またはそれに対する脆弱性です。

## 影響

Prototype Pollution の影響は、さまざまで発生箇所やアプリケーションのロジックに依存します。
最悪の場合ではサーバサイドでの任意コード実行につながることがあり、そのほかの場合では XSS や SQL インジェクションなどの脆弱性にもつながる可能性があります。

## 原因

攻撃者が`__proto__`プロパティなどを経由して、特定のオブジェクトの prototype を操作可能であることが Prototype Pollution の原因です。

## 攻撃手法

実際にオブジェクトの`__proto__`プロパティが操作可能なケースとして、下記 2 つの例を紹介します。

1. オブジェクトに対して merge や clone の操作をするケース
2. `setValue(obj, key, value)`のようにオブジェクトのプロパティを設定するケース

### 1. オブジェクトに対して merge や clone の操作を行うケース

以下の`merge(tgt, src)`は、引数の`tgt`オブジェクトに`src`のプロパティを再帰的にマージします。
この実装では与えられたオブジェクトの`key`の値をチェックしておらず、任意の`key`に対して任意の値を代入できます。(10 行目)

```javascript
function isObject(obj) {
    return obj !== null && typeof obj === 'object';
}

function merge(tgt, src) {
    for (let key in src) {
        if (isObject(tgt[key]) && isObject(src[key])) {
            merge(tgt[key], src[key]);
        } else {
            tgt[key] = src[key];
        }
    }
    return tgt;
}

merge(
    {a: 1, b: 2},
    JSON.parse('{"__proto__": {"polluted": 1}}')
);
const obj = {};
console.log(obj.polluted); // => 1
```

上記の PoC では、引数`src`に`{"__proto__": {"polluted": 1}}`というオブジェクトを渡しています。その値を10 行目で`tgt[__proto__] = {"polluted":1}`のような代入が実行されることで攻撃が成功しています。

### 2. `setValue(obj, key, value)`のようにオブジェクトのプロパティを設定するケース

以下の`setValue(obj, key, value)`は、引数の`obj`オブジェクトに`{key:value}`のプロパティを追加します。また、`key`にチェーン演算子を用いて指定することで深くに位置するプロパティを追加します。

この実装でも引数`key`の値をチェックしておらず、任意の`key`に対して任意の値を代入できます。(13 行目)

```javascript
function isObject(obj) {
    return obj !== null && typeof obj === 'object';
}

function setValue(obj, key, value) {
    const keylist = key.split('.');
    const e = keylist.shift();
    if (keylist.length > 0) {
        if (!isObject(obj[e])) obj[e] = {};

        setValue(obj[e], keylist.join('.'), value);
    } else {
        obj[key] = value;
        return obj;
    }}

setValue({}, "__proto__.polluted", 1);

const a = "";
console.log(a.polluted); // => 1
```

上記の PoC では、引数`key`に`__proto__.polluted`という値を渡しています。その値が再帰的に処理されることで結果的に 13 行目で`obj[__proto__][polluted] = 1`のような代入が実行されることで攻撃が成功しています。

## 事例紹介

- [NVD - cve-2019-7609](https://nvd.nist.gov/vuln/detail/cve-2019-7609)
- [#1106238 Stored XSS via Mermaid Prototype Pollution vulnerability](https://hackerone.com/reports/1106238)
- [#454365 Prototype pollution attack through jQuery $.extend](https://hackerone.com/reports/454365)

## 対策

Prototype Pollution の対策にはいくつか方法があるため、アプリケーションの規模や副作用を考慮して、適切な対策を選択するようにしてください。
ここでは代表的な下記 5 つの方法を紹介します。

- ライブラリを利用する方法
- `Object.freeze`を使用する方法
- `Object`の代わりに`Map`を使う方法
- オブジェクトの作成を`Object.create(null)`で行う方法
- Schema validation of JSON input

### ライブラリを利用する方法

要件を満たす場合、Prototype Pollution の影響を受けずにオブジェクトに対して merge や clone を行うライブラリを利用することで防ぐことができます。

### `Object.freeze`を使用する方法

[Object.freeze()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)を使用して、Object や Object.prototype を変更できないようにすることで、prototype が意図せず汚染されないようにできます。

下記のコードは、実際に`Object.freeze()`を使用して実際に対策できることを示す PoC です。

```javascript
Object.freeze(Object.prototype);
Object.freeze(Object);
({}.__proto__.test = 123);

console.log({}.test); // => undefined
```

なお、注意点として、（上記の PoC を実際に動かしてみてもわかる通り、）`Object.freeze()`によって freeze されたオブジェクトの変更は、特にエラーや例外が起こることなく失敗します。
そのため、本対策を適用し意図しない副作用が発生した場合でも、その発見が困難な場合もあります。
したがって、この対策は（依存ライブラリを含めた）アプリケーションの現状の実装を考慮して、慎重に適用することを推奨します。

### `Object`の代わりに`Map`を使う方法

ES6 以降では、`Object`の代わりに`Map`が利用できます。
`Object`で単純に key/value のデータ構造を利用している場合は、それらを`Map`に置き換えることで、Prototype Pollution を防ぐことができます。

### オブジェクトの作成を`Object.create(null)`で行う方法

`Object.create()`に`null`を渡し、prototype を引き継がないオブジェクトを作成することで、`__proto__`経由で prototype が汚染されることを防ぐことができます。
下記のようなコードで、実際に`__proto__`経由で prototype にアクセスできないことを実際に確認できます。

```javascript
var obj = Object.create(null);

console.log(obj.__proto__ === Object.prototype); // => false
console.log(obj.__proto__); // => undefined
```

もし`{"a": 1, "b": 2}`のようなもともといくつかのプロパティを持つオブジェクトを作成する場合は、下記のように[`Object.assign()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)を併せて利用する必要があります。

```javascript
var obj = Object.assign(Object.create(null), { a: 1, b: 2 });
```

なお、このように作成されたオブジェクトの prototype は`Object.prototype`の参照ではありません。そのため、`hasOwnProperty()`のような`Object.prototype`のメソッドはプロトタイプチェーン経由で呼び出せません。
もし、そのようなメソッドを呼び出したい場合は`Object.prototype.hasOwnProperty`のように明記する必要があります。

```javascript
var obj = Object.create(null);
obj.a = 1;
Object.prototype.hasOwnProperty.call(obj, "a"); // => true
```

`Object.create(null)`による本対策を適用する場合は、このような副作用に留意してください。

### Schema validation of JSON input

[JSON Schema](https://json-schema.org/)では`additionalProperties:false`を指定することで、想定していないプロパティを禁止できます。
適切な JSON Scheme を用いてバリデーションを行うことで Prototype Pollution の対策ができます。

以下は、前述の関数`setValue()`に対し、[ajv](https://ajv.js.org/)を用いて対策する例です。

```javascript
const schema = {
  type: "object",
  properties: {
    // 想定しているプロパティ
  },
  additionalProperties: false,
};
const validate = ajv.compile(schema);

function setValue(obj, key, value) {
  const keylist = key.split(".");
  const e = keylist.shift();
  if (keylist.length > 0) {
    if (!isObject(obj[e])) obj[e] = {};
    setValue(obj[e], keylist.join("."), value);
  } else {
    obj[key] = value;
    if (!validate(obj)) {
      // handling
      throw "Invalid Obj";
    }
    return obj;
  }
}

setValue({}, "__proto__.polluted", 1); // => Exception raised
```

ただし、この場合、`properties`に含んでいないプロパティは一切追加ができなくなることに注意が必要です。
任意のプロパティを受け入れつつ対策をする場合、単に追加される key に悪意のある値が指定されないように制限する対策も有効です。

```javascript
function setValue(obj, key, value) {
  const keylist = key.split(".");
  const e = keylist.shift();
  if (keylist.length > 0) {
    if (!isObject(obj[e])) obj[e] = {};

    if (e !== "__proto__" && e !== "constructor") {
      setValue(obj[e], keylist.join("."), value);
    }
  } else {
    obj[key] = value;
    return obj;
  }
}

setValue({}, "__proto__.polluted", 1);

const a = "";
console.log(a.polluted); // => undefined
```

## 診断方法

### 基本的な診断方法

これまでに説明した Prototype Pollution の基本原理や攻撃手法などを踏まえ、任意のオブジェクトの prototype が不正に変更できないかを検証してください。

### DOM Invader を用いた効率的な診断

DOM Invader は Burp の機能で DOM XSS のテストや`postMessage()`の操作を用いたテストの支援を提供します。この機能を用いることでクライアントサイドの Prototype Pollution の自動検出や手動での検証の補助として利用可能です。

詳しい検証方法は[Testing for client-side prototype pollution](https://portswigger.net/burp/documentation/desktop/tools/dom-invader/prototype-pollution)で丁寧に解説されているので、こちらを参照してください。

## 学習方法/参考文献

- [HoLyVieR/prototype-pollution-nsec18: Content released at NorthSec 2018 for my talk on prototype pollution](https://github.com/HoLyVieR/prototype-pollution-nsec18)
- [Olivier Arteau -- Prototype pollution attacks in NodeJS applications - YouTube](https://youtu.be/LUsiFV3dsK8)
- [Node.js における prototype 汚染攻撃への対策 - SST エンジニアブログ](https://techblog.securesky-tech.com/entry/2018/10/31/)
- [【1 分見て】実例から学ぶ prototype pollution【kurenaif 勉強日記】 - YouTube](https://www.youtube.com/watch?v=qP8ihBctMeY)
- [Prototype pollution: The dangerous and underrated vulnerability impacting JavaScript applications | The Daily Swig](https://portswigger.net/daily-swig/prototype-pollution-the-dangerous-and-underrated-vulnerability-impacting-javascript-applications)
- [BlackFan/client-side-prototype-pollution: Prototype Pollution and useful Script Gadgets](https://github.com/BlackFan/client-side-prototype-pollution)
- [Object.prototype.**proto** - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)
