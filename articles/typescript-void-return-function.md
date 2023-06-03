---
title: TypeScriptで戻り値がvoid型の関数を扱う場合の注意点
emoji: "🈳"
type: "tech"
topics: ["TypeScript", "JavaScript", "void"]
published: true
# published_at: 2023-02-06
---

この記事は、TypeScript で戻り値が void 型の関数を扱う場合の注意点に関する覚書です。

## TL;DR

- TypeScript(JavaScript)における明示的な戻り値を持たない関数は、ランタイムでは`undefined`を返す。
- TypeScript における関数の戻り値としての`void型`は`undefined型`よりも「戻り値の利用を想定していないことを明示できる」点で優位性があるが、知らないとハマりそうな assignability の罠がある。
- 上記の assignability の仕様を把握しておきつつ、以下のように型注釈することが better だと思っている。

| 型注釈をつける関数の種類<br>（戻り値の利用想定） | 戻り値の型注釈                             | 関数型の使用 |
| ------------------------------------------------ | ------------------------------------------ | ------------ |
| 副作用だけの関数                                 | ⭕️ `void`<br>（🔺 `undefined`）           | 避けたい     |
| 戻り値のない return 文で early return する関数   | ⭕️ `T \| void`<br>（🔺 `T \| undefined`） | OK           |

※ 前提として、私は Return Type は積極的に書きたい派です。

## 他言語の関数における戻り値の扱い

いきなり他言語に脱線するのですが、 私が学生の頃に書いたことがある[**Fortran**](https://ja.wikipedia.org/wiki/FORTRAN)では「呼び出されると入力に対し何らかの処理を実行するサブプログラム」のうち、関数（`function`）は値を返すもの、サブルーチン（`subroutine`）は値を返さないもの、といった分類があります。

https://amanotk.github.io/fortran-resume-public/chap07.html

## TypeScript における戻り値の扱い

一方で、**TypeScript**では**実装として明示的な戻り値を持たない関数は void 型の戻り値を持つ**ものとして推論されます。
また、TypeScript（JavaScript）において、これらの関数はランタイムに`undefined`を返します。すなわち**実行時に戻り値を返さない関数は存在しません**。サンプルコードで見てみましょう。

```typescript
// 暗黙的にundefinedを返す関数の戻り値はvoid型と推論される
const f1 = () => {}; // () => void
const f2 = () => {
  return;
}; // () => void
console.log(undefined === f1()); // true
console.log(undefined === f2()); // true

// 明示的にundefinedを返す関数の戻り値はundefined型と推論される
const f3 = () => {
  return undefined;
}; // () => undefined
console.log(undefined === f3()); // true
```

## void 型について

void 型は関数の返り値の型として使われることを想定しており、「なにも返さない」ことを表す型です。
[公式ドキュメント](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#other-important-typescript-types)には void 型について`a subtype of undefined intended for use as a return type.`と記載されており、戻り値の型として使用されることが想定されていることがわかります。

※ 余談ですが 2023/2/1 現在では上記の記載がありますが、厳密には void 型は undefined 型の`supertype`であり、上記は誤記であると思われます。またこれに対する[Pull Request](https://github.com/microsoft/TypeScript-Website/pull/2470)も作成されています。

また、下記サイトから一部引用します。

https://typescriptbook.jp/reference/functions/void-type#undefined%E5%9E%8B%E3%81%A8void%E5%9E%8B%E3%81%AE%E9%81%95%E3%81%84

> TypeScript の型上の意味としては、undefined 型と void 型は同じです。したがって、戻り値の型注釈に undefined を用いることもできます。

ふむ、「undefined 型と void 型は同じ」とあります。

ただし、（個人的な感覚として）それでも**暗黙的に undefinded を返す関数には積極的に void 型で型付けしたい**と考えています。なぜなら、**明示的に戻り値を返す関数と暗黙的に undefined を返す関数は、戻り値の利用を想定しているかの点で本質的には異なるもの**だと考えているからです。
よって、これ以降では**暗黙的に undefined を返す関数の戻り値の型注釈には「なにも返さない（戻り値が利用されることを想定していない）」ことを明示する void 型を極力利用する**という方針のもと、ここからは型注釈をつける関数の種類ごとにみていきます。

## 1. 副作用だけの関数

### 1.1 関数型を使わない場合

これは以下のようなケースです。

```typescript
// OK
const f1 = (): void => {
  console.log("TS is wakaran");
};

// NG
// エラー：　A function whose declared type is neither 'void' nor 'any' must return a value.(2355)
const f2 = (): undefined => {
  console.log("TS is chittomo wakaran");
};

// OK
const f3 = (): undefined => {
  console.log("TS is completely understood");
  return;
};
```

`f2`では「void か any でない戻り値の宣言がある関数は return 文が必須」だとコンパイラに指摘されています。解消するためは`f3`のようにすれば良いですが、undefined 型で型注釈することによって return 文を書く必要が発生するのは本末転倒な気がします（コンパイル後のコードにももちろん残ります）。よって、`f1`のように素直に void 型で注釈するのが良いでしょう。

### 1.2 関数型を使う場合

ここで、一点注意点として、このような関数に[関数型](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions)を用いて型付けをする場合には、いささか慎重に取り扱う必要があります。それは**戻り値が void 型の関数型で型付けした関数は、関数宣言時点で戻り値の方に関わらず Type Error が発生しない**ためです。ただし、そのような関数の戻り値の型は void 型で注釈されているため、戻り値に対して何らかの演算をかけたタイミングで Type Error となりえます。以下のコードでみてみましょう。

```typescript
type F = () => void;
// いずれもコンパイルは通る
const f1: F = () => {
  console.log("TS is yappa wakaran");
};
const f2: F = () => {
  return 100;
};
const f3: F = () => {
  return { foo: 100 };
};

const r3 = f3();
// エラー：　Property 'foo' does not exist on type 'void'.(2339)
r3.foo++;
```

上記のような単純な例であれば特段問題にならないかもしれませんが、関数の利用者が安全にそれ呼び出すためにも、関数実装時点でこのような型不正は防がれるべきです。よって、戻り値が void 型の関数に型注釈をする場合にはそもそも関数型を使用しないほうが安全だと思います。また同様の理由から、[Call Signatures](https://www.typescriptlang.org/docs/handbook/2/functions.html#call-signatures)として`() => void`などで型注釈する際にも注意が必要です。

## 2. 戻り値のない return 文で early return する関数

### 2.1 関数型を使わない場合

前提として、下記のような**early return（早期リターン）を含む関数は、undefined 型との Union Types として型推論されます**~~（void 型じゃないんだ ☺️）~~。[冒頭の結果](#typescriptにおける戻り値の扱い)もあって、この記事の内容を整理するまでは void 型との Union Types と推論されるのかと思っていましたが、関数全体として戻り値が利用される可能性がないケースしか void 型は推論結果に登場しないように思われます。

```typescript
// 関数の型推論: () => "lucky!" | undefined
const judgeFortune = () => {
  if (Math.random() > 0.99) return;
  return "lucky!";
};
```

さて、とはいえこのようなケースでも、型注釈をつけるなら私は void 型との Union Types で型付けします。それは以下の理由からです。

1. early reurn の対象となるケース（条件）においても undefined という実行結果を呼び出し元で利用することを想定しているわけではない。
1. Union Types であれば型チェックでエラーにならない（下記のコードブロックを参照）。

@[codesandbox](https://codesandbox.io/embed/withered-forest-jq8tf6?fontsize=12&hidenavigation=1&theme=dark&view=split&expanddevtools=1codemirror=1&highlights=2)

### 2.2 関数型を使う場合

また、上記のような戻り値が void 型との Union Types になるようなケースでは、関数型を用いて型付けしても問題ありません。最後に下記のサンプルコードを見てみましょう。

```typescript
type F = () => "bravo!" | void;

// OK
const judgeFortune: F = () => {
  if (Math.random() > 0.99) return;
  if (Math.random() > 0.99) return undefined;
  return "bravo!";
};

// NG
const occurMisfortune: F = () => {
  /**
   * エラー:
   * Type '() => "bravo!" | null' is not assignable to type 'F'.
   * Type '"bravo!" | null' is not assignable to type 'void | "bravo!"'.
   * Type 'null' is not assignable to type 'void | "bravo!"'.(2322)
   */
  if (Math.random() > 0.99) return null;
  return "bravo!";
};
```

上記では void 型との Union Types の戻り値をもつ関数型の`F`で関数に型注釈をつけていますが、関数実装時点で void 型の subtype でない null は Type Error として検出できています。

## しめくくり

今回 TypeScript の`void型`について再整理するにあたり、これまで自分の中でもあまり意識していなかったいくつかの項目に気づくことができました。また、私にとっては初めての記事作成となりましたが、web 業界のエンジニアとなって 1 年が経過したこともあり、自身の知見の整理のためにも大小問わずこれからも書いていけたらと思います。本稿に対し、ご指摘やご意見などございましたら、いつでもコメントいただけると嬉しいです 🙋

※ 本記事は以前まとめた[Qiita 記事](https://qiita.com/queuek/items/4f08116000e5b316c846)からの移管先です。

### 参考にさせていただいたもの

https://zenn.dev/estra/articles/typescript-type-set-hierarchy

https://marsquai.com/a70497b9-805e-40a9-855d-1826345ca65f/1dc3824a-2ab9-471f-ad58-6226a37245ce/5b09c1d0-b51f-4594-b651-7a58d86d4e40/

http://nmi.jp/2022-10-17-Understanding-Undefined-And-Null
