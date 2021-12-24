---
title: "Pythonで任意長の整数型がどのように実装されているか"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "翻訳"]
published: false
---

GoogleでソフトウェアエンジニアをしているAlberto Oshiro氏の[How Python Represents Integers using Bignum](https://levelup.gitconnected.com/how-python-represents-integers-using-bignum-f8f0574d0d6b)の翻訳になります。本人のご了承を得て公開しています。感謝。

![](/images/afb263bf68fff2/image.jpeg)
*Photo by [Crissy Jarvis](https://unsplash.com/@crissyjarvis?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/addition?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

C/C++のような低レイヤーのコーディングをしているプログラマは整数型のメモリ使用量を気にしなければいけません。また、オーバーフローを防ぐために変数の取りうる最小・最大値も把握して、`int`で十分なのか、`long`が必要なのか考えなければいけません。

C/C++と違ってPythonには整数値のオーバーフローがないという利点があるので、Pythonプログラマは整数値にどの型を使うか考える必要がないのです。また、Pythonの整数型はとてつもなく大きい数を演算しても精度が失われることはないです。唯一の制限はマシンのメモリ不足(つまりハードウェアによる制限)だけです。

実際に階乗(Factorial)を計算する例をみてみましょう。Pythonでは外部のライブラリを使わずに結果の大きさに関わらず階乗を計算することができます。

```python
def factorial(n):
   if n == 0 or n == 1:
     return 1
   return n * factorial(n-1)
```

`factorial`関数に`231`を入力するとPythonがどれほど大きい整数値を扱えるか分かるでしょう。

```python
>>> factorial(231)
1792233667382633521618843263044232513197622942259968207385215805123682159320161029848328112148883186161436034535802659466205111867109614573242316954383604389464524535467759401326264883566523043560811873179996072188155290081861628010250468430411854935707396605833540921031884571521279145124581094374547412403086564118143957940727734634769439112260383017302489106932716079961487372942529947238400000000000000000000000000000000000000000000000000000000
```

:::message
NOTE: このコード例は整数値の大きさを示すためです。階乗を計算するもっと効率的なアルゴリズムがあります。
:::

## 整数値表現

この記事はもっとも一般的に使われている処理系であるCPythonのでの話になります。他の処理系での整数型の表現は異なるかもしれません。CPythonを使う利点として、CPythonのコードベースがGithubで自由に見れることにあります。

Python3以降では全ての整数値は以下のstructで表現されます。

```cpp
struct _longobject {
   PyObject_VAR_HEAD
   digit ob_digit[1];
};
```

マクロを展開すると以下のようになります。

```cpp
struct {
   sssize_t ob_refcnt;
   struct _typeobject *ob_type;
   ssize_t ob_size;
   uint32_t ob_digit[1];
};
```

最初の二つのフォールドは重要ではないです。`ob_refcnt`はPythonのGCで使われ、`ob_type`は型のIDを表しこの場合は整数になります。

整数値は残り二つのフィールド`ob_digit`, `ob_size`で表します。`ob_digit`は整数値の各桁を格納します。`ob_size`は`db_didit`配列の長さと整数値の符号(正か負)の2つを格納します。

このように整数値の各桁を文字列もしくは配列で表すことを任意精度演算(Bignum Arithmetic)[^1]といいます。

[^1]: [任意精度演算](https://ja.wikipedia.org/wiki/任意精度演算)

$$
Base 2^{30}
$$

`uint32_t`の配列を整数値表現に使うと想定してみましょう。
