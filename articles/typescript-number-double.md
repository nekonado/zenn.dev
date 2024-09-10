---
title: "TypeScriptの数値がIEEE 754倍精度浮動小数点数であることを組み込みオブジェクトから確かめる"
emoji: "🎱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "JavaScript"]
published: true
---

TypeScript（JavaScript）の数値は、IEEE 754 倍精度浮動小数点数（64 ビット）で表現されます。[MDN の Number のページ](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Number#%E6%95%B0%E5%80%A4%E3%81%AE%E3%82%A8%E3%83%B3%E3%82%B3%E3%83%BC%E3%83%87%E3%82%A3%E3%83%B3%E3%82%B0)には以下のように記載されています。

> JavaScript の数値 (Number) 型は IEEE 754 の倍精度 64 ビットバイナリー形式であり、 Java や C# の double のようなものです。つまり、小数値を表しますが、格納される数値の大きさと精度には制限があります。とても簡単に説明すると、IEEE 754 の倍精度数は、3 つの部分を表すのに 64 ビットを使用します。

この計算誤差の有名な例として、TypeScript（JavaScript）では次のような計算結果が得られます。

```typescript
console.log(0.1 + 0.2);
// -> 0.30000000000000004
```

このとき、内部のバイナリ表現では number データが 64 ビットで表現されていることから、0.1 + 0.2 の計算結果が `0.30000000000000004`となる計算誤差が発生しています...とは言っても、やはりそれぞれの小数点数がどのようなビット列で表現されているのか、気になりますよね。そこで、TypeScript（JavaScript）の組み込みオブジェクトを使って、数値がどのように表現されているかを確認してみましょう。

まずは、number データのビット表現を取得する関数を作成します。

```typescript
// ビット列を取得するための関数
function toBinaryString(number) {
  // 8バイト（64ビット）のメモリ領域を確保
  const buffer = new ArrayBuffer(8);
  // メモリ領域を操作するDataViewを作成
  const view = new DataView(buffer);
  // メモリ領域に64ビットの数値をセット
  view.setFloat64(0, number);

  let result = "";

  // 8バイト分のデータを逆順に1バイトずつ読み取り、2進数の文字列に変換
  for (let i = 0; i < 8; i++) {
    // 1バイト（8ビット）を2進数の文字列として取得し、先頭にゼロを埋める
    const bits = view.getUint8(i).toString(2).padStart(8, "0");
    // 逆順にビット列を連結
    result = bits + result;
  }

  // 完成した64ビットの2進数の文字列を返す
  return result;
}

// 0.1のビット表現
const binary_0_1 = toBinaryString(0.1);
console.log(binary_0_1);
// -> 1001101010011001100110011001100110011001100110011011100100111111

// 0.2のビット表現
const binary_0_2 = toBinaryString(0.2);
console.log(binary_0_2);
// -> 1001101010011001100110011001100110011001100110011100100100111111
```

ふむふむ、なにやら複雑なビット列が得られました。このビット列を元に、浮動小数点数を復元する関数を作成して、加算結果がどのようになるか確認してみましょう。

```typescript
// ビット列を浮動小数点数に戻す関数
function toFloat64(binaryString) {
  const buffer = new ArrayBuffer(8);
  const view = new DataView(buffer);

  // 8バイト分のデータを逆順に1バイトずつセット
  for (let i = 0; i < 8; i++) {
    // ビット列を8ビットごとに分割し、2進数から10進数に変換
    const byte = parseInt(binaryString.substring(i * 8, i * 8 + 8), 2);
    view.setUint8(7 - i, byte); // 逆順にセット
  }

  // メモリ領域の64ビットの数値を取得
  return view.getFloat64(0);
}

// 0.1 と 0.2 のビット列を足し合わせる
const sum = toFloat64(binary_0_1) + toFloat64(binary_0_2);

console.log(sum);
// -> 0.30000000000000004
console.log(sum === 0.1 + 0.2);
// -> true
```

ビット列を元に戻す関数を使って、0.1 と 0.2 のビット列を足し合わせた結果を再計算してみました。結果は、やはり`0.30000000000000004`となりました 🎉

本記事では、組み込みオブジェクトを使って数値のビット列を取得し、ビット列を元に戻す関数を作成して、数値がどのように表現されているかを確認しました。計算結果がどのようになるか気になる方は、ぜひお手元の JavaScript ランタイムで試してみてください。
