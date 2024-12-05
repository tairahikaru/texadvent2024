---
title: TeX で限界を突破する
description: |
  TeX で限界を突破しよう！
author: tairahikaru
lang: ja
date: 2024-12-06
lastmodified: '2024-12-06'
source: https://github.com/tairahikaru/texadvent2024/blob/main/index.md
license: https://creativecommons.org/licenses/by-sa/4.0/
comment: https://github.com/tairahikaru/texadvent2024/issues/1
layout: mylayout
tag: TeX. Overflow
documentclass: bxjsarticle
classoption: pandoc
toc: true
colorlinks: true
header-includes: |
  \usepackage{newunicodechar,xparse}
  \newunicodechar{⟨}{\meta}
  \NewDocumentCommand\meta{u{⟩}}{$\langle\mbox{#1}\rangle$}
---

{% raw %}

これは「[TeX ＆ LaTeX Advent Calendar 2024](https://adventar.org/calendars/10647)」の 6 日目の記事である。
昨日は鴎海ねこさんの「[unicode-math ユーザーのための Unicode 文字入力補助](https://github.com/kmi-ne/TeX-MyTools/tree/main/unicode-math-snippets)」だった。明日のことはわからない。

この記事は [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)、記事中のコードはパブリックドメイン・無保証とする。

# 限界を越える方法

TeX で内部整数として扱えるのは絶対値が 2^31 未満の数である。
2^31 を 16 進数で表記すると 7FFFFFFF で、絶対値がこれを超える数を扱おうとするとエラーが出る。

```tex
\count255="7FFFFFFF
\showthe\count255 % -> 2147483647.
\count255="80000000 % -> ! Number too big.
```

ところが、実は次のようにするとこの限界を突破できる。

```tex
\count255="7FFFFFFF
\advance\count255 1
\showthe\count255
```

上のコードは実行してもエラーが出ない。
`\showthe` の結果は `-2147483648.` となり、オーバーフローしているのが見て取れる。

はじめてこの挙動を発見したときは 「TeX のバグを発見してしまった……」と思ったのだが、どうやら（やはり）バグではないようだ。
TeX のバグを疑われて報告されたもののうちバグではないとされたものを記載している “[Not a bug in original TeX](https://tug.org/texmfbug/nobug.html)” の “[B506 (section 1238): bogus display of bogus dimen, and other overflow](https://tug.org/texmfbug/nobug.html#overflow)” に同様の記述があり（つまりそもそも既知だった）、それによると
> the lack of overflow checking is pervasive in TeX, and is a deliberate choice by Knuth

とのことである。

[そこ](https://tug.org/texmfbug/nobug.html#overflow)で報告されているのは dimen の例で、dimen においては限界を超えてもすぐにはオーバーフローしないことがうかがえる：

```tex
\dimen0="40000000 sp % -> ! Dimension too large.
\dimen0="3FFFFFFF sp
\advance\dimen0 \dimen0
\showthe\dimen0 % -> 32767.99997pt.
```

なんと、こうして得られる長さ 32767.99997 pt は、plain TeX や LaTeX などで定義されている TeX で扱える最大の長さ `\maxdimen` の 16383.99998 pt よりも大きい。
`\maxdimen` は plain TeX などでは大体次のように定義されている：

```tex
\dimendef\maxdimen10
\maxdimen=16383.99998pt
```

この 16383.99998 pt は sp に直すと 1073741823 sp で、これは 2^31-1 ではなく 2^30-1 である。

そもそも、長さに関しては比較的簡単に限界を越えられる：

```tex
\setbox0=\hbox{\hskip16383.99998pt \hskip16383.99998pt}
\showthe\wd0 % -> 32767.99997pt.
```

わたしは常々どうして数値の上限は \"7FFFFFFF なのに長さの上限は \"3FFFFFFF sp なのか不思議だったのだが、このように box 組み立て時に上限を超えてしまいうるからだとすれば納得である。

# 活用法

## Overfull を表示しない

マクロの内部などで、overfull の警告をなにがあっても表示したくない場合があり、そういうときには `\hfuzz` や `\vfuzz` を `\maxdimen` に設定するというのが普通である。
これらのプリミティヴは overfull の大きさが設定された長さ「未満」である場合に overfull の警告を抑止する。
つまり、overfull がちょうど `\maxdimen` であった場合には抑止できないことになる。

そもそも、上で述べたように長さに関しては `\maxdimen` を越えることも十分起こりうる。
そういう場合にも警告を抑えたいのであれば、
```tex
\hfuzz="3FFFFFFF sp
\advance\hfuzz \hfuzz
\advance\hfuzz 1sp
\vfuzz="3FFFFFFF sp
\advance\vfuzz \vfuzz
\advance\vfuzz 1sp
```

とすれば、`\maxdimen` の長さ overfull した場合であっても警告が出ない。
唯一の例外は overfull した長さがちょうど \"7FFFFFFF sp であった場合だが、これはそもそも TeX で扱える長さの上限を超えているので、ちょうど `\maxdimen` だけ overfull する場合よりも珍しいだろう。

## 整数を読みとばす

ZR さんのブログ「[整数を読み飛ばす、続き (1)](https://zrbabbler.hatenablog.com/entry/20110403/1301854441)」で与えられているお題だが、要するに
- `\gobblenum⟨数値⟩` を先頭で何回か展開すると空になるようなマクロ `\gobblenum` を定義せよ
- ただし `⟨数値⟩` は TeX で扱える数値の表現であればどんなものでも構わない

ということで、もっというと
```tex
\def\gobblenum{%
  \begingroup\afterassignment\endgroup
  \count255=}
```
を（先頭）完全展開可能に実装せよということである。

[ZR さんのブログ](https://zrbabbler.hatenablog.com/entry/20110403/1301854441)では ε-TeX でなくても使える方法として
```tex
\def\gobblenum{%
   \expandafter\fi\ifnum-"7FFFFFFF<}
```
というものが紹介され、以下の欠点があると述べられている：

1. `⟨数値⟩` がちょうど `-"7FFFFFFF` のとき失敗する
2. 数値読み取り中に `\else` や `\fi` があった場合失敗する

このうち、1 については限界を突破した内部整数を使えば解決できる。
すなわち
```tex
\count255="7FFFFFFF
\advance\count255 1
\def\gobblenum{%
  \expandafter\fi\ifnum\count255<}
```
とすれば、この条件判断は `⟨数値⟩` がどのようなものであってもそれが TeX の整数として認められるものである限り真になる。
唯一の例外は `⟨数値⟩` がこうして作られた本来 TeX で扱えないオーバーフローした -2147483648 の場合であるが、それは仕様外としてもよいだろう。

ところで、ε-TeX の場合には [ZR さんのブログ](https://zrbabbler.hatenablog.com/entry/20110406/1302109724)にあるように `\parshapeindent` を用いることができるので、必然的にこの方法を使わざるを得ないのは ε-TeX でない場合ということになる。
そうすると貴重なレジスタをこんなくだらないトリックのために 1 つ消費することになる（[BXjscls](https://github.com/zr-tex8r/BXjscls) にならって `\fontdimen` の活用を試してみたのだが、`! Dimension too large.` に引っかかった）。

以降 `\count255` は `\gobblenum` 中で使うために独占的に使用できるものとする。

---

ついでに 2 についても触れておく。
詳細は [ZR さんのブログ](https://zrbabbler.hatenablog.com/entry/20110403/1301854441)や[ここ](https://blog.miz-ar.info/2018/06/tex-noexpand-and-inserted-relax/)をみて欲しいが、ともかく
```tex
\immediate\write16{\iftrue\expandafter\fi\ifnum0=0\fi}
```
とすると `\relax` が表示されるのが問題だということである。
つまり、
```tex
\immediate\write16{%
  \iftrue\gobblenum0\fi}
```
とすると `\relax` が出てきてしまう。

実はこのケースに対処すること自体は簡単で、次のようにすればよい：

```tex
\count255="7FFFFFFF
\advance\count255 1
\def\gobblenum{%
  \expandafter\fi% (A)
  \expandafter\fi\ifnum\count255<%
  \iftrue}% (B)
```

こうすると数字の読み取り中に現れた `\fi` や `\else` は (B) の `\iftrue` に対応するものとして扱われて展開する。
そんなことをして平気なのかと思うかもしれないが、`\fi` などが現れたということはその前でなんらかの条件判断が真になったということであり、であれば数値読み取り中に `\fi` などを展開するのは count レジスタなどではむしろ平常である。
その前にあるはずの条件判断には (A) の `\fi` が対応する。

さて、もちろんこれが失敗するケースを作るのも簡単である：

```tex
\immediate\write16{%
  \iftrue
    \iftrue\gobblenum0\fi% (C)
  \fi}% (D)
```

こうすると、(C) の `\fi` は (B) に対応して読み飛ばされてくれるが、(D) の `\fi` のところで `\relax` が挿入される。
このケースに対処するにはやはり `\gobblenum` 中の `\expandafter\fi` と `\iftrue` を増やせばよく、それを失敗させるには数値読み取り中の `\fi` とかを増やせばよい……、というわけで永遠に終わらない。

この問題を完璧に解決することはできないが、ある程度であれば `\gobblenum` 中にまず通常使われないであろう数だけ `\expandafter\fi` と `\iftrue` を入れる、という戦略が考えられる。

とりあえず実験のために `\manyX⟨制御綴⟩{⟨数値⟩}{⟨トークン列⟩}` とすると `⟨トークン列⟩` を `⟨数値⟩` 回繰り返したトークン列に展開するマクロとして `⟨制御綴⟩` を定義する、というマクロを作った。
たとえば `\manyX\xxx{3}{x}` とすると `\def\xxx{xxx}` としたのと同じになる。
以降はこれを使ったコードを書くが、その実装を理解する必要はない。

```tex
\def\manyX#1#2#3{\begingroup
  \count255=#2%
  \edef\x##1##2\fi##3##4{\noexpand\fi\endgroup
    \def##3{%
      ##1\ifodd\count255 ##4\fi}}%
  \divide\count255 2
  \ifnum\count255=0
    \x{}%
  \else
    \manyX\y{\count255}{#3#3}%
    \expandafter\x\expandafter{\y}%
  \fi{#1}{#3}}
```

まずは `\gobblenum` の定義を次のように直す：

```tex
\count255="7FFFFFFF
\advance\count255 1
\def\gobblenum{%
  \manyexpandafterfi
  \expandafter\fi\ifnum\count255<%
  \manyiftrue}
```

この上で、以下の `100` の部分の数字を色々変えてみる。

```tex
\manyX\manyexpandafterfi{100}{\expandafter\fi}
\manyX\manyiftrue{100}{\iftrue}
\immediate\write16{%
  \iftrue\iftrue\iftrue\iftrue
    \gobblenum0%
  \fi\fi\fi\fi}
```

わたしの環境では `10000` にしたあたりで `! TeX capacity exceeded, sorry [expansion depth=10000].` に引っかかった。
この限界を超えたければ texmf.cnf を編集すればよいが、ともかく 10000 重にネストした条件判断は想定されてないようなので、例えば半分の 5000 とかにしてみる。

```tex
\manyX\manyexpandafterfi{5000}{\expandafter\fi}
\manyX\manyiftrue{5000}{\iftrue}
\immediate\write16{%
  \iftrue\iftrue\iftrue\iftrue
    \gobblenum0%
  \fi\fi\fi\fi}
```

うまくいってそうだが 1 つ問題がある：

```tex
\immediate\write16{%
  \gobblenum0\gobblenum0}% -> ! TeX capacity exceeded, sorry [expansion depth=10000].
```

というわけで平方をとって結局 100 ぐらいでよいだろう。
そうすると `\gobblenum` の数値読み取り中に 99 個以上の別の `\gobblenum` や合計 100 個を越える `\fi` または `\else` が出現した場合は失敗するが、そうでない場合にはおおむねうまくいく：

```tex
\manyX\manyexpandafterfi{100}{\expandafter\fi}
\manyX\manyiftrue{100}{\iftrue}
\manyX\manymanyiftrue{101}{\iftrue}
\manyX\manymanyfi{101}{\fi}
\immediate\write16{%
  \manymanyiftrue
    \gobblenum0\gobblenum0\gobblenum0%
  \manymanyfi}% -> \relax
```

# 結論

安易に TeX 言語プログラミングで巨大な数を使うと、エラーが出ずにオーバーフローするかもしれないので気をつけよう。

{% endraw %}

