---
title: TeX で限界を突破する（その 2）
description: |
  \ifcase で限界を突破しよう！
author: tairahikaru
lang: ja
date: 2024-12-26
source: https://github.com/tairahikaru/texadvent2024/blob/main/ifcase.md
license: https://creativecommons.org/licenses/by-sa/4.0/
comment: https://github.com/tairahikaru/texadvent2024/issues/2
layout: mylayout
tag: TeX
documentclass: bxjsarticle
classoption: pandoc
toc: true
colorlinks: true
header-includes: |
  \usepackage{newunicodechar,xparse}
  \newunicodechar{⟨}{\meta}
---

{% raw %}

「[TeX & LaTeX Advent Calendar 2024](https://adventar.org/calendars/10647)」の 6 日目に[記事](https://tairahikaru.github.io/texadvent2024/)を書いたあとに、ふと気になったことがある。
その記事では `\advance` を用いると限界を突破できることを説明したが、`\ifcase` 中に `\or` がたくさん（\"7FFFFFFF 個より多く）でてくるとどうなるのだろうか。

ということで試してみた。

ちなみに 25 日の記事は ZR さんの「[重点解説！ イマドキの LaTeX の“命令フック機能”](https://qiita.com/zr_tex8r/items/70ba9e910c8d265307fa)」だ。

# `\manyX` で突破する試み

まずは、[先日の記事](https://tairahikaru.github.io/texadvent2024/#%E6%95%B4%E6%95%B0%E3%82%92%E8%AA%AD%E3%81%BF%E3%81%A8%E3%81%B0%E3%81%99)で使った `\manyX`（をなるべく省メモリになるように再実装したつもりのもの）を使って \"7FFFFFFF 個の `\or` を含むマクロを定義し、実行することを試みる：

```tex
\catcode`\{=1 \catcode`\}=2 \catcode`\#=6
\def\manyX#1#2#3{\begingroup
  \count255=#2%
  \toks0{%
    \edef\x##1{%
      \ifnum\count255<2
        \the\toks2
      \else
        \divide\count255 2
        \the\toks0{##1##1}%
      \fi
      \ifodd\count255 ##1\fi}\x}%
  \toks2{\the\toks4\expandafter{\ifnum`}=0 \fi}%
  \toks4{\endgroup\def#1}%
  \the\toks0\ifnum`{=0 \fi{#3}}}
\manyX\tmp{"7FFFFFFF}{\or}
\immediate\write16{\ifcase-1\tmp\fi}
```

このコードを実行すると、

```
! TeX capacity exceeded, sorry [main memory size=5000000]
```
となる。

texmf.cnf を編集する代わりにコマンドライン引数を用いて `tex -ini -cnf-line=main_memory=5000000000` とした（`main_memory` は initex でしか反映されない）ところ、`Ouch---my internal constants have been clobbered!---case 14` という見慣れないエラーが出た。
[Stack Exchange のこの回答](https://tex.stackexchange.com/a/67016)によれば 12,435,455 より大きい値は指定できないということである。
16 進数で 7FFFFFFF は 10 進数で 2,147,483,647 であるので、12,435,455 のメモリではどう考えても入りきらない。

# それでも限界を突破したい

諦めるのはまだ早い。
メモリに載り切らないのであれば、メモリに載せずに実行すればよいのだ！

具体的には

```tex
\ifcase-1
⟨\or\or\or\or…
\or がたくさん並ぶ⟩
\fi
```
という TeX ファイルを作れば、たくさんある `\or` をメモリに載せずにつど読み込んで読み飛ばしていけるはずである。

というわけでそんなファイルを生成するためのスクリプトを Lua で書いた（TeX で書く気は起きなかった）。
LuaTeX がインストールされているのであれば texlua も使えると考えられるので、それで実行してもよい。

<div markdown="block" class="warn">

4 GB を超えるサイズの TeX ファイル（正確には 4,295,396,905 byte）が生成されるので注意。

実行は環境にもよるだろうが、数秒で終わる。

</div>

```lua
f = io.open('manytilde.tex', 'w')
f:write('\\catcode`\\~13 \\let~\\or')
f:write('\\ifcase-1\n')

-- '~' の繰り返し回数
n = 0xFFFFFFFF

l = 10000
r = n % l
n = n // l
l = l - r
line = '\n'

while r > 0 do
  line = '~' .. line
  r = r - 1
end
f:write(line)

while l > 0 do
  line = '~' .. line
  l = l - 1
end

l = 200
r = n % l
n = n // l
lines = ''
l = l - r

while r > 0 do
  lines = lines .. line
  r = r - 1
end
f:write(lines)

while l > 0 do
  lines = lines .. line
  l = l - 1
end

print(n)
while n > 0 do
  f:write(lines)
  n = n - 1
  if n % 100 == 0 then
    print(n)
  end
end

f:write('\\immediate\\write16{-1}')
f:write('~')
f:write('\\immediate\\write16{0}')
f:write('~')
f:write('\\else')
f:write('\\immediate\\write16{else}')
f:write('\\fi')
f:write('\\end')
f:close()
```

さて、こうして作られる manytilde.tex の中身は次のようになっている：

```tex
\catcode`\~13 \let~\or\ifcase-1
⟨~~~~~~~~…
~ が合計 "FFFFFFFF 個適宜改行を挟みつつ並ぶ⟩
\immediate\write16{-1}~\immediate\write16{0}~\else\immediate\write16{else}\fi\end
```

間に改行を入れてるのは `! Unable to read an entire line---bufsize=200000.` に引っかかるため。

これを実行すると、わたしの環境では 5 分足らずで終了し、コンソールに `-1` が出力される。
すなわち `\immediate\write16{-1}` が実行されたということである。
つまり `\or` の数え上げが overflow して `\ifcase-1` で `\else` 以外の節が選択されたということになる。

# 結論

`\ifcase` で負の数でも条件分岐したいときは、次のようにすべし：

```tex
\ifnum\count255<0
  \ifcase-\count255
    ⟨条件分岐の中身（負の場合）⟩
  \fi
\else
  \ifcase\count255
    ⟨条件分岐の中身（非負の場合）⟩
  \fi
\fi
```

この記事は [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)、コード部分はパブリックドメイン・無保証である。

{% endraw %}
