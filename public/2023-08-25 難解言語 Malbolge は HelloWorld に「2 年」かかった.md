---
title: 難解言語 Malbolge は HelloWorld に「2 年」かかった
tags:
  - C++
  - HelloWorld
  - Esolang
  - ビームサーチ
  - Malbolge
private: false
updated_at: '2023-09-10T21:11:08+09:00'
id: 1e5f4fb54916c6fed483
organization_url_name: null
slide: false
ignorePublish: false
---
# TL; DR
こちらが難解プログラミング言語 **Malbolge** で HelloWorld を行うプログラムになります。~~プログラムとはいったい・・・~~

```
(=<`$9]7<5YXz7wT.3,+O/o'K%$H"'~D|#z@b=`{^Lx8%$Xmrkpohm-kNi;gsedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543s+O<oLm
```

# Malbolge
**Malbolge** は 1998 年 Ben Olmstead により開発された難解プログラミング言語です。オリジナルのホームページはとっくに消滅していますが、幸いにもアーカイブは残っていました。

https://web.archive.org/web/20031202211441/http://www.mines.edu:80/students/b/bolmstea/malbolge/

Malbolge は難解プログラミング言語の中でも特に**難解であること自体を目的に開発された**、文字通り**地獄みたいな**言語です。

> Malbolge was truly created with the idea that programming should be hard. It should be as close to the Infernal as a programming language possibly can be. It will continue to evolve over time, as newer, more twisted minds attack this problem.
>
> 拙訳：Malbolge という言語は実に、プログラミングは難しくあるべきという思想のもと作り出されました。Malbolge は、一つのプログラミング言語として可能な限り、地獄に近いものであるべきなのです。Malbolge は進化し続けます――より新しく、よりひねくれた頭脳の持ち主たちが、この Malbolge という名の問いに挑み続けるたびに。
>
> 出典：https://web.archive.org/web/20031202211441/http://www.mines.edu:80/students/b/bolmstea/malbolge/

Malbolge という名前自体、**地獄の第八圏・悪の嚢 (Malebolge)** に由来するものです。その実態はともに有名な難解言語である INTERCAL と BrainF**c を悪魔合体したものになっています。以下詳しく紹介していきますが、厳密な仕様と実装に関しては[付録 A](https://qiita.com/reika727/items/1e5f4fb54916c6fed483#付録-a-malbolge-のオリジナル仕様書) と[付録 B](https://qiita.com/reika727/items/1e5f4fb54916c6fed483#付録-b-malbolge-のオリジナル実装) を参照してください。

## 環境
Malbolge は**三進コンピュータ**上で動作します。つまり一般的なコンピュータが 0, 1 の二つの値を取る「ビット」を操作するのに対し、Malbolge 仮想マシンは 0, 1, 2 の三つの値を取る「**トリット**」を操作します。

Malbolge マシンでは 10 トリットを 1 ワードとします。つまり、1 ワードは 0 から $3^{10} - 1 = 59048$ までの値を取ります。あとで出てくる三種のレジスタもそれぞれ 1 ワードで、メモリも $3^{10} = 59049$ ワードからなります。

## 実行
:::note warn
Malbolge は**オリジナル仕様とオリジナル実装に大きな乖離があります**。このことに関して [Malbolge の作者である Ben Olmstead にインタビューを行った記事](https://esoteric.codes/blog/interview-with-ben-olmstead#documentation)があるのですが、どうも仕様と実装の乖離に関しては**知ったうえでわざと放置してる**みたいです。えぇぇ・・・

これより先、仕様と実装で食い違っている部分や、C 言語によるオリジナル実装が未定義動作を起こす状況に関してはその都度脚注で指摘していくので、興味のある人だけ読んでってください。
:::

先ほどお見せしたとおり、典型的な Malbolge コードはこんな感じの見た目をしています。

```
(=<`$9]7<5YXz7wT.3,+O/o'K%$H"'~D|#z@b=`{^Lx8%$Xmrkpohm-kNi;gsedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543s+O<oLm
```

まずはこれらの文字（の ASCII コード）でメモリを 0 番地から順に埋めていきます[^note-on-meminit]。空白文字は無視されます。またメモリが 59048 番地までしかないことから、ソースコードが 59049 文字を超える場合は Malbolge 仮想マシンは異常終了します。

[^note-on-meminit]: 仕様によると空白文字でない文字は XLAT1 変換（後述）したものが `'j'`, `'i'`, `'*'`, `'p'`, `'<'`, `'/'`, `'v'`, `'o'` のどれかにならなければならず、そうでない場合 Malbolge 仮想マシンは異常終了することになっています。ただしオリジナル実装においては「空白文字でも ASCII 印字可能文字でもない文字」、つまりヌル文字などの制御文字に遭遇した場合はエラーになることなくメモリに書き込まれます。まあ Malbolge に限らずソースコードというものはふつう印字可能文字だけで書かれるものですから、ソースコードに制御文字が含まれる可能性は想定していなかったのかもしれません。

逆にソースコードが 59049 文字に満たない場合は以下に示す **crazy 演算**[^note-on-crazy]で残りのメモリを埋めていきます。

[^note-on-crazy]: 「crazy 演算」という名前が一般的に用いられてはいるのですが、オリジナルの仕様書やソースコード内にはこの呼び名は見当たりません。仕様書ではただ単に "op"（恐らくは "operation" の略）と呼ばれています。crazy 演算という呼び名の初出は不明です。

```
     |    x
_____|_0__1__2_
   0 | 1  0  0
y  1 | 1  0  2
   2 | 2  2  1
```

これは二つのワード `x`, `y` をとって一つのワードを返す二項演算です。`x`, `y` の各桁から上の表に従って計算結果の桁を求めていきます。またこの表には**規則性は特にありません**。単にテキトーに決められただけのものです。

例として `crazy(100, 500)` を計算してみます。$100_{(10)}$ と $500_{(10)}$ は十桁の三進表記ではそれぞれ $0000010201_{(3)}$ と $0000200112_{(3)}$ となるので、`crazy(100, 500)` は $1111201212_{(3)} = 29696_{(10)}$ となります。

この crazy 演算を用いて、`memory[i] = crazy(memory[i - 1], memory[i - 2])`[^note-on-crazyinit]といった具合に未初期化メモリをアドレスの若い順に初期化していきます。メモリの初期化が完了したらいよいよプログラムを実行していきます。

[^note-on-crazyinit]: このとき、Malbolge のオリジナル実装ではソースコードの文字数が 2 文字未満だと配列外参照による未定義動作が発生しますが、仕様では特に禁止されていません。

先ほどもチラッと出てきましたが、Malbolge には三種類のレジスタがあります。機械語をいじったことのある方は見覚えのある名前だと思います。

- **A レジスタ**：演算結果の記録に使われる。Accumulator の略。
- **C レジスタ**：実行中の命令へのポインタ。Code の略。
- **D レジスタ**：メモリ上のデータへのポインタ。Data の略。

これらのレジスタは全て 0 で初期化されます。その後、以下の流れでプログラムを実行していきます。

### 1. 命令のフェッチ
まずは「`C` レジスタに記録されているアドレスに位置しているデータ」を取得します。以下、アセンブリ言語に倣ってこれを `[C]` と表記します（`D` についても同様）。

いきなりわかりづらいので具体例を考えてみます。前述のようにレジスタは 0 で初期化されるので、Malbolge が実行されてから最初の命令フェッチでは 0 番地のデータ、すなわちソースコードの一文字目（の ASCII コード）が取り出されることになります。

ここで取り出されたデータは空白文字以外の ASCII 印字可能文字でなければなりません。そうでない場合プログラムは停止することになっています[^note-on-halting]。

[^note-on-halting]: 仕様書には `[C]` が空白文字以外の ASCII 印字可能文字でない場合はプログラムを停止すると書かれているのですが、実装の方ではなぜか無限ループに陥ります。

取り出したデータはつづけて **XLAT1 変換**[^note-on-xlat1]されます。まず取り出したデータから 33 を引きます。`[C]` に入っているのは「空白文字以外の ASCII 印字可能文字」、つまり ASCII コードでいえば 33 から 126 のどれかですから、33 を引くことで 0 から 93 に収まることになります。これにさらに `C` を足し、94 で割った剰余を下のテーブルへのインデックスとして用います。

[^note-on-xlat1]: この名前は仕様書に記されたものではなく、実装中で用いられている変数名から私が勝手に名付けたものです（後で出てくる XLAT2 変換についても同様）。"XLAT" というのは恐らく機械語の XLAT 命令が由来です。

```
+b(29e*j1VMEKLyC})8&m#~W>qxdRp0wkrUo[D7,XTcA"lI.v%{gJh4G\-=O@5`_3i<?Z';FNQuY]szf$!BS/|t:Pn6^Ha
```

もはや言うまでもないことでしょうが、この変換にもやっぱり規則性などはありません。

### 2. 命令の実行
Malbolge には 8 種類の命令があります。上の変換表から得られた文字に従って以下の命令[^note-on-ops]のどれかを実行します。

[^note-on-ops]: オリジナル実装では **`'<'` 命令と `'/'` 命令が逆になっています**。ガバガバすぎんだろ・・・

- `'j'` の場合：`[D]` を `D` に代入する。
- `'i'` の場合：`[D]` を `C` に代入する。すなわちジャンプ命令に相当する。
- `'*'` の場合：`[D]` を右に一ケタ循環シフト[^note-on-shift]し、`A` にも代入する。
- `'p'` の場合：`A` と `[D]` に `crazy(A, [D])` を代入する。
- `'<'` の場合：標準入力から一文字受け取り、その ASCII コードを `A` に代入する。ただし `EOF` が入力された場合は $2222222222_{(3)} = 59048_{(10)}$ を代入する。
- `'/'` の場合：`A` を ASCII コードに変換し[^convert-to-ascii]標準出力に出力する。[^note-on-bug]
- `'v'` の場合：プログラムを終了する。
- `'o'` の場合：何もしない。すなわち NOP 命令に相当する。
- それ以外：何もしない。つまり実質 `'o'` に同じ。

[^note-on-shift]: 「三進法での」シフトであることに注意してください。

[^convert-to-ascii]: `A` は 0 から 59048 までの値を取りますが、オリジナル実装ではこれを直接 `putc` に渡しています。そして `putc` の仕様では `int` 型引数として受け取った文字コードを `unsigned char` に変換して出力に書き込むことになっています。よって C 言語の厳密な仕様に沿って出力命令の挙動を解釈すると以下のようになります。まず `int` 型は最低でも 16 ビットあることが仕様により定められていますが、仮に `int` 型がちょうど 16 ビットであった場合 -32768 から 32767 までしか入らないことになります。したがって `A` が 32767 を超えている場合、`putc` の引数として渡される値は**実装定義**です。逆に `int` 型が 17 ビット以上ある場合は 0 から 59048 までの数がすべて入るので、この場合は `A` を `putc` の引数として渡したときには値は変化しません。その後あらためて `unsigned char` への変換が行われます。ここでは符号なし整数への変換が行われるので、`A` を $2^\mathrm{CHAR\\_BIT}$ で割った最小非負剰余が最終的に文字コードとして用いられることになります。まあ、とはいっても今やほとんどの環境で `int` 型は 32 ビットでしょうし、`char` 型は 8 ビットでしょうから、出力命令の挙動は「`A` を 256 で割った剰余を文字コードとして出力する」と解釈していいと思います。

[^note-on-bug]: Malbolge のオリジナル実装では出力命令にバグがあります。この命令の実装はこのようになっているのですが、

    ```c
    #if '\n' != 10
            if ( x == 10 ) putc( '\n', stdout ); else
    #endif
            putc( a, stdout );
    ```

    これは恐らく以下を意図したものだと思われます。

    ```diff_c
    #if '\n' != 10
    -       if ( x == 10 ) putc( '\n', stdout ); else
    +       if ( a == 10 ) putc( '\n', stdout ); else
    #endif
            putc( a, stdout );
    ```

    とは言ったものの、そもそも `'\n' != 10` が真になるような処理系が稀でしょう。これに関しては今回の調査で私も初めて知ったのですが、実は C 言語の仕様では実行時に使用する文字セットは ASCII と決められているわけではないそうです。例えば実行環境で使用されている文字セットが EBCDIC の場合などは `'\n' != 10` が真になります。そのような処理系で入力命令よりも先に出力命令を実行すると未初期化変数の参照による未定義動作が発生します。それにしてもさっきから未定義動作ばっか踏んでるくせになんでこんな稀なケースだけは想定しているんだ・・・（Malbolge の開発には IBM PC でも使ってたのか？）

### 3. メモリの暗号化
命令を実行した次は **XLAT2 変換**によって `[C]` を暗号化します。`[C]` から 33 を引き、下のテーブルへのインデックスとして用います[^note-on-jmp]。

[^note-on-jmp]: 直前に実行された命令が `'i'` 命令である場合 `C` が書き換わっているので `[C]` が 33 から 126 の範囲にない可能性があります。この場合オリジナル実装では配列外参照による未定義動作が発生しますが、仕様では特に禁止されていません。

```
5z]&gqtyfr$(we4{WP)H-Zn,[%\3dL+Q;>U!pJS72FhOA1CB6v^=I_0/8|jsb9m<.TVac`uY*MK'X~xDl}REokN:#?G"i@
```

変換結果を `[C]` に代入します。つまり、Malbolge は `C` 番地のメモリを毎回いちいち暗号化してしまうのです。

### 4. レジスタのインクリメント
最後に `C` レジスタと `D` レジスタをインクリメントします。ただしインクリメントした結果 59049 に達した場合はまた 0 に戻ってきます。このあとまた命令フェッチから再開します。

# Malbolge による HelloWorld
このとおり Malbolge は地獄みたいな言語なので、まともなプログラムが発表されるまで 2 年[^note-on-two]の歳月を要しました。~~だろうね~~

[^note-on-two]: Malbolge が発表された正確な日付も HelloWorld プログラムが発表された正確な日付も今となってはわかりませんが、[Malbolge のオリジナルのホームページのアーカイブ](https://web.archive.org/web/20031202211441/http://www.mines.edu:80/students/b/bolmstea/malbolge/)の更新履歴より、Malbolge は遅くとも 1998 年の 4 月には公開されていたことがわかります。また HelloWorld プログラムを作り出した Andrew Cooke のブログの[最古のアーカイブ](https://web.archive.org/web/20001018215726/http://www.andrewcooke.free-online.co.uk/andrew/writing/malbolge.html)は 2000 年 10 月 18 日のものでした。また、[別のプログラムの作成に成功した人物のブログ](https://web.archive.org/web/20020223231155/http://www.antwon.com/index.php?p=234)によると Cooke による HelloWorld が公開されたのは「2000 年の 4 月まで」だそうです。ですので、確証はありませんが、恐らく 2000 年ごろが Cooke による HelloWorld が公開された時期だと思われます。

Malbolge による HelloWorld は Andrew Cooke により開発されました。といっても手作業で作り出したわけではなく、**ビーム探索アルゴリズム**によってようやく見つけ出すことに成功したようです。

・・・と、ここまでは Wikipedia にも書いてあります。しかしながら、Cooke が実際にビーム探索に用いたソースコードの実物がいくら探しても見つかりません。これでは Malbolge による HelloWorld が本当にビーム探索で見つけ出されたのかわかりません。そこで調査を続けた結果、Cooke 本人によるアルゴリズムの解説記事のアーカイブが見つかりました。

https://web.archive.org/web/20200807093255/http://acooke.org:80/malbolge.html

そこで**衝撃の事実**が発覚しました。

> i deleted the lisp code when updating the OS on my computer. before that i had generated a properly punctuated "Hello world", but never saved the code. so i guess this will remain the only non-trivial malbolge program....
>
> 拙訳：コンピュータの OS をアップデートしたときに、**（HelloWorld の探索に用いた）LISP コードを削除しました**。その前にはきちんと句読点がついた "Hello world" も生成できていたのですが、その時のコードも保存してませんでした。なのでこの Malbolge コードがただ一つの価値ある Malbolge コードであり続けると思います・・・
>
> 出典：https://web.archive.org/web/20200807093255/http://acooke.org:80/malbolge.html

<font color=#55C500><b>(　ﾟдﾟ)</b></font>

# 作ってみよう（血涙）
なんということでしょう。Cooke が作り出したプログラムはとっくの昔に消滅していました。なので自分で作ってみるしかありません。本当にビーム探索アルゴリズムで HelloWorld プログラムを見つけ出せるか試してみましょう。

ここで鍵になってくるのが先ほども登場した XLAT1 変換です。脚注でも書いていましたが Malbolge ソースコードは各文字を XLAT1 変換したものが `'j'`, `'i'`, `'*'`, `'p'`, `'<'`, `'/'`, `'v'`, `'o'` のどれかにならなければなりません。例えば記事冒頭でお見せした HelloWorld プログラムを XLAT1 変換するとこのようになります。

```
jpp<jp<pop<<jo*<popp<o*p<pp<pop<pop<jijoj/o<vvjpopoopo<ojo/ovooooooooooooooooooooooooooooooooooooooooooooooooooo*p<v*<*
```

こうしてみると、Malbolge ソースコードも結局は BrainF**c のように各種命令の羅列として表現できることがわかります。

よってこのような方策が考えられます。まずこのように、全メモリ領域が初期化されていない Malbolge 仮想マシンを考えます。レジスタは仕様通り全て 0 で初期化します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/507500/c4f564c2-7447-6966-b17c-c16eff62a906.png)

Malbolge 仮想マシンは `C` 番地の命令を実行するのでした。よって最初は 0 番地の命令が実行されるのですが、ここは当然まだ初期化されていません。そこで、**この 0 番地に 8 種の命令それぞれを入れてみます。**

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/507500/50e255a4-626b-fb36-5e66-dc8b5d585af1.png)

するとこのように 0 番地が無事初期化されたマシンが 8 個できます。このそれぞれが 0 番地のメモリの値に基づいて計算を続けていきます。

あとはこの繰り返しです。つまり、**初期化されていないメモリに遭遇してしまうたびにそこを 8 種の命令のそれぞれで初期化し、仮想マシンの状態を分岐させていくのです**。専門用語を用いればこれは**幅優先探索**にあたります。

## ビーム探索アルゴリズム
ですがこの方法には問題があります。毎回毎回 8 回も分岐していれば計算すべき状態数は指数関数的に増えていってしまいます。そこで用いるのが**ビーム探索アルゴリズム**です。

|![](https://camo.qiitausercontent.com/ccd80b676ab3d8b51562378070c441a38b983c63/68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f7461696c2d69736c616e642f77617465722d6a75672d70726f626c656d2f6d61737465722f696d6167652f6265616d2d7365617263682e6a7067)|
|:-:|
|出典：[勝手に幅優先探索と最良優先探索とビーム・サーチでWater Jug Problemを解き直してみた](https://qiita.com/tail-island/items/b873b4890353b50f2eac)|

この図のように、ビーム探索アルゴリズムでは探索の各深さで各ノードの**スコア**を計算し、上位のものだけを探索対象として残します。こうすることによって計算量の増加を大幅に抑えることができるのです。

:::note info
ただしビーム探索アルゴリズムは幅優先探索の上位互換ではありません。ビーム探索アルゴリズムは「枝刈り」を行っているので、場合によっては解が見つからなかったり、見つかったとしても最適解ではなかったりする場合があります。

またビーム探索アルゴリズムはスコア上位の何件かのみを選び出すわけですが、生き残れるノードのうち「同率最下位」のものは計算結果が偏るのを防止するため乱択するのが普通です。つまりビーム探索アルゴリズムは実行ごとに結果が異なる可能性があります。

幅優先探索とは、それぞれのメリット・デメリットを理解したうえで使い分けることが大切ということですね。
:::

このような方法で Cooke は "Hello world" と出力してくれるプログラムを探索しました（ただし計算時間を短縮するため大文字・小文字の区別は行いませんでした）。また「スコア」には「出力された文字数 × 10 - 状態遷移回数」を用いました。こうすることによって状態遷移回数を抑えつつ目的のプログラムを見つけ出すことができます。

## 実装
というアルゴリズムを Cooke は LISP で実装したのですが、前述の通りそのコードは現存しません（半ギレ）。なので私が再現してみました。ただし私は LISP が使える**宇宙人**ではないので、C++ で実装しました。

|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/507500/c857162c-c0c6-bf40-f495-7fd0ebb1632a.png)|
|:-:|
|LISP の~~公式~~キャラクター。「LISP が分からない人からは LISP 使いがこう見えている」というジョークがある。|

実際に作ったプログラムがこちらです。

https://github.com/reika727/MalbolgeHelloWorld

そして実行結果がこちらです。

```console
$ make test
./malbolge-hello.out
GENERATION #1
        GENERATION SIZE: 1
        BEST RESULT    :
        BEST SCORE     : 0

# （中略）

GENERATION #52
        GENERATION SIZE: 10000
        BEST RESULT    : Hello WorlD
        BEST SCORE     : 59

        FINAL RESULT   : Hello WorlD
        FINAL SCORE    : 59
        CODE           : (=<`#9]76ZY32V6/S3,Pq)M'&Jk#Gh~D1#"!~}|{z(Kw%utsVqpihml>jibgJedFFaDY^Wi
```

おまけで作ったインタプリタで動作を確かめてみましょう。

```console
$ cat hello.tmp.mb
(=<`#9]76ZY32V6/S3,Pq)M'&Jk#Gh~D1#"!~}|{z(Kw%utsVqpihml>jibgJedFFaDY^Wi
$ __tests__/malbolge.out hello.tmp.mb
Hello WorlD
```

というわけで、Cooke は嘘をついていなかったことがわかりました。また Cooke がこのアルゴリズムを組んだ 2000 年（推定）は計算資源等もいろいろ厳しかったようですが、23 年経った現在ではわずか **15 秒**で探索が終了しました。感動的ですね～。

## ついでに
先ほどお話ししたようにビーム探索アルゴリズムは実行ごとに結果が変わる場合があります。私が作ったプログラムもそうです。なので `'i'` 命令（ジャンプ命令）を用いるプログラムが最終的に採用された場合など**クソ長い**コードが出力されます。まあ、もちろんそれでもちゃんと動きますが(ﾄﾞﾔｧ

```console
FINAL RESULT   : HellO woRld
FINAL SCORE    : 54
CODE           : (=<`#9]76ZY{y1UvS3ss*N(o&J$#Gh~De{A.~}|{z\Kw%$t"Vqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:98765u3sr0`.-Jm*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%1#AR~,|{)]
```

# おわりに
ところで、

> incidentally, i've come to hate malbolge.
>
> 出典：https://web.archive.org/web/20200807093255/http://acooke.org:80/malbolge.html

Cooke は Malbolge のことがキライになったそうです。さもありなん。

# 付録 A: Malbolge のオリジナル仕様書
[オリジナルのホームページのアーカイブ](https://web.archive.org/web/20031202211441/http://www.mines.edu:80/students/b/bolmstea/malbolge/)からダウンロードできるソースコードに付属しているものです。

<details><summary>仕様書</summary><div>

```
Malbolge
'98, Ben Olmstead

I hereby relenquish any and all copyright on this language,
documentation, and interpreter; Malbolge is officially public domain.

------------------------------------------------------------------------

                                Malbolge
                           '98, Ben Olmstead

Introduction
^^^^^^^^^^^^

It was noticed that, in the field of esoteric programming languages,
there was a particular and surprising void: no programming language
known to the author was specifically designed to be difficult to program
in.

Certainly, there were languages which were difficult to write in, and
far more were difficult to read (see: Befunge, False, TWDL, RUBE...).
But even INTERCAL and BrainF***, the two kings of mental torment, were
designed with other goals: INTERCAL to have nothing in common with any
major programming language, and BrainF*** to be a very tiny, yet still
Turing-complete, language.

INTERCAL's constructs are certainly tortuous, but they are all too
flexible; you can, for instance, quite easily assign any number to a
variable with a single statement.

BrainF*** is lacking the flexibility which is INTERCAL's major weakness,
but it fails in that its constructs are far, far too intuitive.
Certainly, there are only 8 instructions, none of which take any
arguments--but it is quite easy to determine how to use those
instructions.  Subtract 8 from the current number?  With a simple
'--------' you are done!  This kind of simple answer was unacceptable to
the author.

Hence the author created Malbolge.  It borrows from machine, BrainF***,
and tri-INTERCAL, but put together in a unique way.  It was designed to
be difficult to use, and so it is.  It is designed to be
incomprehensible, and so it is.

So far, no Malbolge programs have been written.  Thus, we cannot give an
example.

"Malbolge" is the name of Dante's Eighth Circle of Hell, in which
practitioners of deception (seducers, flatterers, simonists, thieves,
hypocrites, and so on) spend eternity.


Environment
^^^^^^^^^^^

In many languages, the environment is easy to understand.  In Malbolge,
it is best to understand the runtime environment before you ever see a
command.

The environment is, roughly, that of a primitive trinary CPU.  Both code
and data share the same space (the machine's memory segment), and there
are three registers.  Machine words are ten trits (trinary digits) wide,
giving a maximum possible value of 59048 (all numbers are unsigned).
Memory space is exactly 59049 words long.

The three registers are A, C, and D.  A is the accumulator, used for
data manipulation.  A is implicitly set to the value written by all
write operations on memory.  (Standard I/O, a distinctly non-chip-level
feature, is done directly with the A register.)

C is the code pointer.  It is automatically incremented after each
instruction, and points the instruction being executed.

D is the data pointer.  It, too, is automatically incremented after each
instruction, but the location it points to is used for the data
manipulation commands.

All registers begin with the value 0.

When the interpreter loads the program, it ignores all whitespace.  If
it encounters anything that is not one of an instruction and is not
whitespace, it will give an error, otherwise it loads the file, one non-
whitespace character per cell, into memory.  Cells which are not
initialized are set by performing op on the previous two cells
repetitively.


Commands
^^^^^^^^

When the interpreter tries to execute a program, it first checks to
see if the current instruction is a graphical ASCII character (33
through 126).  If it is, it subtracts 33 from it, adds C to it, mods it
by 94, then uses the result as an index into the following table of 94
characters:

            +b(29e*j1VMEKLyC})8&m#~W>qxdRp0wkrUo[D7,XTcA"lI
            .v%{gJh4G\-=O@5`_3i<?Z';FNQuY]szf$!BS/|t:Pn6^Ha

It then checks it against the characters listed below, and performs an
appropriate action.

If the result is not one of the characters listed below, it is treated
as a nop.  If the original character is not graphic ASCII, the program
is immediately ended.

When the interpreter parses the input file, it checks each non-
whitespace character with the process above.  If any result is not one
of the eight characters below, the file will be rejected.

After the instruction is executed, 33 is subtracted from the instruction
at C, and the result is used as an index in the table below.  The new
character is then placed at C, and then C is incremented.

            5z]&gqtyfr$(we4{WP)H-Zn,[%\3dL+Q;>U!pJS72FhOA1C
            B6v^=I_0/8|jsb9m<.TVac`uY*MK'X~xDl}REokN:#?G"i@

j
  sets the data pointer to the value in the cell pointed to by the
  current data pointer.

i
  sets the code pointer to the value in the cell pointed to be the
  current data pointer.

*
  rotates the trinary value of the cell pointed to by D to the right 1.
  The least significant trit becomes the most significant trit, and all
  others move one position to the left.

p
  performs a tritwise "op" on the value pointed to by D with the
  contents of A.  The op (don't look for pattern, it's not there) is:

            | A trit:
    ________|_0__1__2_
          0 | 1  0  0
      *D  1 | 1  0  2
     trit 2 | 2  2  1



Di-trits:
    00 01 02 10 11 12 20 21 22

00  04 03 03 01 00 00 01 00 00
01  04 03 05 01 00 02 01 00 02
02  05 05 04 02 02 01 02 02 01
10  04 03 03 01 00 00 07 06 06
11  04 03 05 01 00 02 07 06 08
12  05 05 04 02 02 01 08 08 07
20  07 06 06 07 06 06 04 03 03
21  07 06 08 07 06 08 04 03 05
22  08 08 07 08 08 07 05 05 04

<
  reads an ASCII value from the stdin and converts it to Trinary, then
  stores it in A.  10 (line feed) is considered 'newline', and
  2222222222t (59048 dec.) is EOF.

/
  converts the value in A to ASCII and writes it to stdout.  Writing
  10 is a newline.

v
  indicates a full stop for the machine.

o
  does nothing, except increment C and D, as all other instructions do.


Turing-Completeness
^^^^^^^^^^^^^^^^^^^

Though I have not proven it, I _think_ Malbolge to be Turing-complete.
To be Turing-complete, there must be some data construct which can be
used to do any mathematical calculation.  I believe that using *p in
various clever ways on the tritwords can fulfill this requirement.

Turing-completeness also requires three code constructs: sequential
execution (which Malbolge obviously has), repetition (provided by the
i and, indirectly, j instructions), and conditional-execution (provided,
I believe, by self-modifying code and altering i destinations).

I do have my doubts, particularly about data constructs, but I *think*
this works...


Appendix: Trinary Conversion Table
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Trinary to ASCII to decimal to hex table, provided, strangely enough,
for the convenience of Malbolge programmers.

00000 NUL 000 00    01012   032 20    02101 @ 064 40    10120 ` 096 60
00001 SOH 001 01    01020 ! 033 21    02102 A 065 41    10121 a 097 61
00002 STX 002 02    01021 " 034 22    02110 B 066 42    10122 b 098 62
00010 ETX 003 03    01022 # 035 23    02111 C 067 43    10200 c 099 63
00011 EOT 004 04    01100 $ 036 24    02112 D 068 44    10201 d 100 64
00012 ENQ 005 05    01101 % 037 25    02120 E 069 45    10202 e 101 65
00020 ACK 006 06    01102 & 038 26    02121 F 070 46    10210 f 102 66
00021 BEL 007 07    01110 ' 039 27    02122 G 071 47    10211 g 103 67
00022 BS  008 08    01111 ( 040 28    02200 H 072 48    10212 h 104 68
00100 HT  009 09    01112 ) 041 29    02201 I 073 49    10220 i 105 69
00101 LF  010 0a    01120 * 042 2a    02202 J 074 4a    10221 j 106 6a
00102 VT  011 0b    01121 + 043 2b    02210 K 075 4b    10222 k 107 6b
00110 FF  012 0c    01122 , 044 2c    02211 L 076 4c    11000 l 108 6c
00111 CR  013 0d    01200 - 045 2d    02212 M 077 4d    11001 m 109 6d
00112 SO  014 0e    01201 . 046 2e    02220 N 078 4e    11002 n 110 6e
00120 SI  015 0f    01202 / 047 2f    02221 O 079 4f    11010 o 111 6f
00121 DLE 016 10    01210 0 048 30    02222 P 080 50    11011 p 112 70
00122 DC1 017 11    01211 1 049 31    10000 Q 081 51    11012 q 113 71
00200 DC2 018 12    01212 2 050 32    10001 R 082 52    11020 r 114 72
00201 DC3 019 13    01220 3 051 33    10002 S 083 53    11021 s 115 73
00202 DC4 020 14    01221 4 052 34    10010 T 084 54    11022 t 116 74
00210 NAK 021 15    01222 5 053 35    10011 U 085 55    11100 u 117 75
00211 SYN 022 16    02000 6 054 36    10012 V 086 56    11101 v 118 76
00212 ETB 023 17    02001 7 055 37    10020 W 087 57    11102 w 119 77
00220 CAN 024 18    02002 8 056 38    10021 X 088 58    11110 x 120 78
00221 EM  025 19    02010 9 057 39    10022 Y 089 59    11111 y 121 79
00222 SUB 026 1a    02011 : 058 3a    10100 Z 090 5a    11112 z 122 7a
01000 ESC 027 1b    02012 ; 059 3b    10101 [ 091 5b    11120 { 123 7b
01001 FS  028 1c    02020 < 060 3c    10102 \ 092 5c    11121 | 124 7c
01002 GS  029 1d    02021 = 061 3d    10110 ] 093 5d    11122 } 125 7d
01010 RS  030 1e    02022 > 062 3e    10111 ^ 094 5e    11200 ~ 126 7e
01011 US  031 1f    02100 ? 063 3f    10112 _ 095 5f

11202 128 80    12221 160 a0    21010 192 c0    22022 224 e0
11210 129 81    12222 161 a1    21011 193 c1    22100 225 e1
11211 130 82    20000 162 a2    21012 194 c2    22101 226 e2
11212 131 83    20001 163 a3    21020 195 c3    22102 227 e3
11220 132 84    20002 164 a4    21021 196 c4    22110 228 e4
11221 133 85    20010 165 a5    21022 197 c5    22111 229 e5
11222 134 86    20011 166 a6    21100 198 c6    22112 230 e6
12000 135 87    20012 167 a7    21101 199 c7    22120 231 e7
12001 136 88    20020 168 a8    21102 200 c8    22121 232 e8
12002 137 89    20021 169 a9    21110 201 c9    22122 233 e9
12010 138 8a    20022 170 aa    21111 202 ca    22200 234 ea
12011 139 8b    20100 171 ab    21112 203 cb    22201 235 eb
12012 140 8c    20101 172 ac    21120 204 cc    22202 236 ec
12020 141 8d    20102 173 ad    21121 205 cd    22210 237 ed
12021 142 8e    20110 174 ae    21122 206 ce    22211 238 ee
12022 143 8f    20111 175 af    21200 207 cf    22212 239 ef
12100 144 90    20112 176 b0    21201 208 d0    22220 240 f0
12101 145 91    20120 177 b1    21202 209 d1    22221 241 f1
12102 146 92    20121 178 b2    21210 210 d2    22222 242 f2
12110 147 93    20122 179 b3    21211 211 d3
12111 148 94    20200 180 b4    21212 212 d4
12112 149 95    20201 181 b5    21220 213 d5
12120 150 96    20202 182 b6    21221 214 d6
12121 151 97    20210 183 b7    21222 215 d7
12122 152 98    20211 184 b8    22000 216 d8
12200 153 99    20212 185 b9    22001 217 d9
12201 154 9a    20220 186 ba    22002 218 da
12202 155 9b    20221 187 bb    22010 219 db
12210 156 9c    20222 188 bc    22011 220 dc
12211 157 9d    21000 189 bd    22012 221 dd
12212 158 9e    21001 190 be    22020 222 de
12220 159 9f    21002 191 bf    22021 223 df

```
</div></details>

# 付録 B: Malbolge のオリジナル実装
こちらもオリジナルのホームページのアーカイブからダウンロードできるものです。

<details><summary>ソースコード</summary><div>

```c
/* Interpreter for Malbolge.                                          */
/* '98 Ben Olmstead.                                                  */
/*                                                                    */
/* Malbolge is the name of Dante's Eighth circle of Hell.  This       */
/* interpreter isn't even Copylefted; I hereby place it in the public */
/* domain.  Have fun...                                               */
/*                                                                    */
/* Note: in keeping with the idea that programming in Malbolge is     */
/* meant to be hell, there is no debugger.                            */
/*                                                                    */
/* By the way, this code assumes that short is 16 bits.  I haven't    */
/* seen any case where it isn't, but it might happen.  If short is    */
/* longer than 16 bits, it will still work, though it will take up    */
/* considerably more memory.                                          */
/*                                                                    */
/* If you are compiling with a 16-bit Intel compiler, you will need   */
/* >64K data arrays; this means using the HUGE memory model on most   */
/* compilers, but MS C, as of 8.00, possibly earlier as well, allows  */
/* you to specify a custom memory-model; the best model to choose in  */
/* this case is /Ashd (near code, huge data), I think.                */

#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <malloc.h>
#include <string.h>

#ifdef __GNUC__
static inline
#endif
void exec( unsigned short *mem );

#ifdef __GNUC__
static inline
#endif
unsigned short op( unsigned short x, unsigned short y );

const char xlat1[] =
  "+b(29e*j1VMEKLyC})8&m#~W>qxdRp0wkrUo[D7,XTcA\"lI"
  ".v%{gJh4G\\-=O@5`_3i<?Z';FNQuY]szf$!BS/|t:Pn6^Ha";

const char xlat2[] =
  "5z]&gqtyfr$(we4{WP)H-Zn,[%\\3dL+Q;>U!pJS72FhOA1C"
  "B6v^=I_0/8|jsb9m<.TVac`uY*MK'X~xDl}REokN:#?G\"i@";

int main( int argc, char **argv )
{
  FILE *f;
  unsigned short i = 0, j;
  int x;
  unsigned short *mem;
  if ( argc != 2 )
  {
    fputs( "invalid command line\n", stderr );
    return ( 1 );
  }
  if ( ( f = fopen( argv[1], "r" ) ) == NULL )
  {
    fputs( "can't open file\n", stderr );
    return ( 1 );
  }
#ifdef _MSC_VER
  mem = (unsigned short *)_halloc( 59049, sizeof(unsigned short) );
#else
  mem = (unsigned short *)malloc( sizeof(unsigned short) * 59049 );
#endif
  if ( mem == NULL )
  {
    fclose( f );
    fputs( "can't allocate memory\n", stderr );
    return ( 1 );
  }
  while ( ( x = getc( f ) ) != EOF )
  {
    if ( isspace( x ) ) continue;
    if ( x < 127 && x > 32 )
    {
      if ( strchr( "ji*p</vo", xlat1[( x - 33 + i ) % 94] ) == NULL )
      {
        fputs( "invalid character in source file\n", stderr );
        free( mem );
        fclose( f );
        return ( 1 );
      }
    }
    if ( i == 59049 )
    {
      fputs( "input file too long\n", stderr );
      free( mem );
      fclose( f );
      return ( 1 );
    }
    mem[i++] = x;
  }
  fclose( f );
  while ( i < 59049 ) mem[i] = op( mem[i - 1], mem[i - 2] ), i++;
  exec( mem );
  free( mem );
  return ( 0 );
}

#ifdef __GNUC__
static inline
#endif
void exec( unsigned short *mem )
{
  unsigned short a = 0, c = 0, d = 0;
  int x;
  for (;;)
  {
    if ( mem[c] < 33 || mem[c] > 126 ) continue;
    switch ( xlat1[( mem[c] - 33 + c ) % 94] )
    {
      case 'j': d = mem[d]; break;
      case 'i': c = mem[d]; break;
      case '*': a = mem[d] = mem[d] / 3 + mem[d] % 3 * 19683; break;
      case 'p': a = mem[d] = op( a, mem[d] ); break;
      case '<':
#if '\n' != 10
        if ( x == 10 ) putc( '\n', stdout ); else
#endif
        putc( a, stdout );
        break;
      case '/':
        x = getc( stdin );
#if '\n' != 10
        if ( x == '\n' ) a = 10; else
#endif
        if ( x == EOF ) a = 59048; else a = x;
        break;
      case 'v': return;
    }
    mem[c] = xlat2[mem[c] - 33];
    if ( c == 59048 ) c = 0; else c++;
    if ( d == 59048 ) d = 0; else d++;
  }
}

#ifdef __GNUC__
static inline
#endif
unsigned short op( unsigned short x, unsigned short y )
{
  unsigned short i = 0, j;
  static const unsigned short p9[5] =
    { 1, 9, 81, 729, 6561 };
  static const unsigned short o[9][9] =
    {
      { 4, 3, 3, 1, 0, 0, 1, 0, 0 },
      { 4, 3, 5, 1, 0, 2, 1, 0, 2 },
      { 5, 5, 4, 2, 2, 1, 2, 2, 1 },
      { 4, 3, 3, 1, 0, 0, 7, 6, 6 },
      { 4, 3, 5, 1, 0, 2, 7, 6, 8 },
      { 5, 5, 4, 2, 2, 1, 8, 8, 7 },
      { 7, 6, 6, 7, 6, 6, 4, 3, 3 },
      { 7, 6, 8, 7, 6, 8, 4, 3, 5 },
      { 8, 8, 7, 8, 8, 7, 5, 5, 4 },
    };
  for ( j = 0; j < 5; j++ )
    i += o[y / p9[j] % 9][x / p9[j] % 9] * p9[j];
  return ( i );
}

```
</div></details>
