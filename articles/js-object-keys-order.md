---
title: "Object.keys()が出力する配列の順序"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [JavaScript]
published: true
---

JavaScriptには[Object.keys()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys)というJavaScriptオブジェクトのプロパティキー一覧を配列で返す関数があります。以下のような挙動をします。

```js
const o = { a: 1, b: 2 };
const keys = Object.keys(o);
console.log(keys); // ['a', 'b']
```

同じオブジェクトをこの関数に渡すとこの関数が返すプロパティキー配列の順序は常に同じようなのですが、仕様でどう定義されているのだろうと気になったので、ECMAScript仕様を読む練習がてら、調べてみました。

参照した仕様は記事作成時点で最新標準のECMAScript 2024[^1]です。

## 結論

結論として、`Object.keys()` はオブジェクトのプロパティキーを以下の順序で列挙します

1. 配列のインデックス（array index）として有効なプロパティキーをインデックス的に昇順で列挙
2. 1に該当しなかった文字列型のプロパティキーをプロパティが作成された順序で列挙

## 例

今回`Object.keys`の挙動確認に使う例です。

```js
const o = {
  foo: "foo",
  [Symbol("symbol1")]: "symbol symbol1",
  2: "two",
  0: "zero",
  bar: "bar",
  1: "one",
  1.2: "one point two",
};

console.log(Object.keys(o));
// [ '0', '1', '2', 'foo', 'bar', '1.2' ]
//   -------------  -------------------
//   上記1.に該当　　　 上記2.に該当
```

この例の結果を見てみると、以下のような点に気づきます。

- 整数で指定したプロパティ（`0`, `1`, `2`）が配列上の順序では先になっていて、それらに関しては数値的な昇順の順序になっている（宣言順ではない）
- 整数でない数値で指定したプロパティは宣言順に順序になっている
- シンボル型のプロパティが出力結果に含まれていない

この辺りの挙動が仕様でどのように定義されているかを確認します。

## Object.keys()の仕様を追っていく

Object.keysの仕様[^2]は以下のように定義されています

> 1. Let obj be ? ToObject(O).
> 2. Let keyList be ? EnumerableOwnProperties(obj, key).
> 3. Return CreateArrayFromList(keyList).

*1.* の `ToObject()` はObject Typeの値を渡した場合、そのまま渡した値を返すだけです。*2.* の `EnumerableOwnProperties(obj, key)` という、プロパティキーを列挙していそうなところを見ていきます。

:::details ToObject? EnumerableOwnProperties?
これらは*Abstract Operation*と呼ばれるもので、ECMAScriptの仕様の中だけで利用される関数のようなものです。

- [ECMAScript仕様を読むのに必要な知識 - ダイジェスト版](https://speakerdeck.com/syumai/ecmascriptshi-yang-wodu-munonibi-yao-nazhi-shi-daiziesutoban?slide=24)
- [5.2.1 Abstract Operations](https://tc39.es/ecma262/#sec-algorithm-conventions-abstract-operations)
  :::

`EnumerableOwnProperties()` の定義[^3]の最初はこちらになっています。

> 1. Let ownKeys be ? O.\[\[OwnPropertyKey\]\]().
> 2. ...

*1.* でオブジェクトの内部的なメソッド `[[OwnPropertyKeys]]()` を呼び出していて、これがオブジェクトのプロパティキー一覧を返してくれます。

`[[OwnPropertyKeys]]()`の定義[^4]では `OrdinaryOwnPropertyKeys(O)`を呼び出しているだけですので、そちらを見ていきます。

`OrdinaryOwnPropertyKeys(O)`の定義[^5]は以下になっています。

> 1. Let keys be a new empty List.
> 2. For each own property key P of O such that P is an array index, in ascending numeric index order, do
>    a. Append P to keys.
> 3. For each own property key P of O such that P is a String and P is not an array index, in ascending chronological order of property creation, do
>    a. Append P to keys.
> 4. For each own property key P of O such that P is a Symbol, in ascending chronological order of property creation, do
>    a. Append P to keys.
> 5. Return keys.

*2.* では配列のインデックスとなれるプロパティキーの一覧を結果の配列に詰めています。*P is an array index* でプロパティキーが配列のインデックスとなれるものを指しています（具体的には$0 <= P <= 2^{32} - 2$[^6]）。
*3.* で*2.* に該当しなかった文字列型のプロパティキーの一覧を配列に詰めています。*ascending chronological order*というところが宣言順（オブジェクトに追加された順序）を意味していそうです。
*4.* ではシンボル型のプロパティキーの一覧を配列に詰めています。
*5.* で最後に *2.*,*3.*,*4.* のところで要素を詰め込んだ配列を返します。
「整数で指定したプロパティが配列上の順序では先になっていて、それらに関しては数値的な昇順の順序になっている」,「整数でない数値で指定したプロパティは宣言順に順序になっている」という挙動を仕様上見つけることができました。

もう1つの「シンボル型のプロパティが出力結果に含まれていない」というところはどこかを見ていきます。
ここまで `Object.keys()` >  `EnumerableOwnProperties()` > `[[OwnPropertyKeys]]()` > `OrdinaryOwnPropertyKeys(O)`と関数呼び出しの流れを見てきました。`OrdinaryOwnPropertyKeys(O)`が返す配列まで確認できましたので、呼び出し元の`EnumerableOwnProperties()`[^3]に戻ります。続きのロジック以下になっています。

> 1. Let ownKeys be ? O.\[\[OwnPropertyKeys\]\]().
> 2. Let results be a new empty List.
> 3. For each element key of ownKeys, do
>     a. If key is a String, then
>     ...略...
>     a. Append key to results.

*3.a. If key is a String* のところで、*文字列型だったら*結果の配列に詰め込むという処理になっているので、シンボル型はここで結果から省かれていることになります。
この定義だと宣言時には数値型でプロパティを指定したもの（`0`, `1.2`など）が結果から省かれてしまうのではと思ってしまうのですが、オブジェクト宣言時の処理で数値型で指定したプロパティキーは文字列型に変換されている[^7]ので、数値型で指定したプロパティもオブジェクト上では文字列型で扱われています。

## 問題になるケース

プロパティキーが配列の要素として利用可能か可能でないか（$2^{32}$付近、十進数で10桁）で挙動が変わるため、10桁のIDのようなものをオブジェクトのプロパティキーとすると思わぬ挙動になってしまうことがあったみたいです。
[Object.keys() が返す配列の順序における数値キーの昇順には上限がある](https://kaminashi-developer.hatenablog.jp/entry/2024/06/04/120000)

## 以前は仕様上順序が保証されていなかった

ECMAScript 5.1の `Object.keys()` の仕様ではfor-inと同じ順序にしなければならないという記述がありますが、for-inの仕様には順序の定義はありません。
ECMAScript 2015で `[[OwnPropertyKeys]]()`の順序が定義[^8]され、`integer index`として扱えるプロパティキーは数値的な昇順、それ以外の文字列はプロパティの作成順になりました。
しかし、`EnumerableOwnNames (O)`[^9]の *6.* では以下のように`[[Enumerate]]`内部メソッドと同様の順序になるように並べ替えを行うようになっています。

> 2\. Let ownKeys be O.\[\[OwnPropertyKeys\]\]().
> ...略...
> 6\. Order the elements of names so they are in the same relative order as would be produced by the Iterator that would be returned if the [[Enumerate]] internal method was invoked on O.

`[[Enumerate]]()`内部メソッドの仕様[^10]では以下のように列挙の順序は定められていません。

> The mechanics and order of enumerating the properties is not specified

なので、`EnumerableOwnNames()`を利用している、`Object.keys()`, `Object.values()`, `Object.entries()`, `JSON.stringify()`, `JSON.parse()`については順序が仕様上決まっていませんでした。[互換性の問題を考慮してこうなっていた](https://stackoverflow.com/questions/30076219/does-es6-introduce-a-well-defined-order-of-enumeration-for-object-properties/30919039#30919039:~:text=While%20ES6%20/%20ES2015%20adds%20property%20order%2C%20it%20does%20not%20require%20for%2Din%2C%20Object.keys%2C%20or%20JSON.stringify%20to%20follow%20that%20order%2C%20due%20to%20legacy%20compatibility%20concerns.)そうです。

ECMAScript 2020で[for-inの順序を仕様で定義する](https://github.com/tc39/proposal-for-in-order)ということになり、`EnumerableOwnNames()`から上記*6.* の並び替えの処理がなくなりました。
これによって`Object.keys()`等の出力するプロパティキーの順序が仕様上定義されるようになりました。

## 最初に列挙される数値範囲の変更

最初に列挙される数値の対象がECMAScript2015以降[integer index](https://tc39.es/ecma262/2024/#integer-index)だったものが、ECMAScript 2019で[array index](https://tc39.es/ecma262/2024/#array-index)に変更されています。
integer indexは $0$ ~ $2^{53}-1$ の範囲、array indexは $0$ ~ $2^{32-2}$ の範囲、という数値の範囲の違いなのですが、なぜその変更が入ったのかという理由はこちらの記事に記載されていました。

https://zenn.dev/pixiv/articles/b6393aad961221

## さいごに

JavaScriptの挙動で何かわからなかったときに一時ソースである仕様書を読めると、自信を持って説明ができるので良いと思います。今回は仕様の中でも割と読みやすいの部分だった気がします。

## 参考

- [ECMAScript仕様の読み方ガイド 〜比較演算子編〜](https://speakerdeck.com/syumai/how-to-read-ecmascript-spec)
- [ECMAScript仕様を読むのに必要な知識 - ダイジェスト版](https://speakerdeck.com/syumai/ecmascriptshi-yang-wodu-munonibi-yao-nazhi-shi-daiziesutoban)
- https://scrapbox.io/esspec/
- [Understanding the ECMAScript spec, part 1](https://v8.dev/blog/understanding-ecmascript-part-1)
- [Does ES6 introduce a well-defined order of enumeration for object properties?](https://stackoverflow.com/a/30919039/6253973)

[^1]: https://tc39.es/ecma262/2024/

[^2]: https://tc39.es/ecma262/2024/#sec-object.keys

[^3]: https://tc39.es/ecma262/2024/#sec-enumerableownproperties

[^4]: https://tc39.es/ecma262/2024/#sec-ordinary-object-internal-methods-and-internal-slots-ownpropertykeys

[^5]: https://tc39.es/ecma262/2024/#sec-ordinaryownpropertykeys

[^6]: https://tc39.es/ecma262/2024/#array-index

[^7]: https://tc39.es/ecma262/2024/#sec-object-initializer-runtime-semantics-evaluation の `LiteralPropertyName : NumericLiteral`のところで`ToString()`によって文字列に変換されている

[^8]: https://262.ecma-international.org/6.0/#sec-ordinary-object-internal-methods-and-internal-slots-ownpropertykeys

[^9]: https://262.ecma-international.org/6.0/#sec-enumerableownnames

[^10]: https://262.ecma-international.org/6.0/#sec-ordinary-object-internal-methods-and-internal-slots-enumerate
