---
title: 【ネタ】include文を悪用してgccに"6那由多"行のエラーを吐かせる
tags:
  - C
  - GCC
  - ネタ
private: false
updated_at: '2022-07-01T20:58:21+09:00'
id: fbcc8749bcd6ae381deb
organization_url_name: null
slide: false
ignorePublish: false
---
# 「おまじない」include文

C言語は初心者泣かせの言語です。まず初心者の第一歩、"Hello World"からしてよくわからないところがいっぱいです。

```c
#include <stdio.h> // これなに？
int main()         // intって？mainって？
{
    puts("Hello World");
}
```

そして初心者泣かせということは、つまり教える側泣かせでもあります。込み入った議論になってしまうことを避けるため、難しいところは「おまじない」とされてしまうことも少なくありません。**#include**はその最たる例と言えるでしょう。

しかしinclude文は決して難しいことをしているわけではありません。誤解を恐れずにざっくりと説明してしまえば、

**#include \<foo\>と書いてある場所にファイルfooの中身を丸ごと貼り付ける**

だけのことです[^directives]。それでは実例を見てみましょう。ここに**header.h**と**source.c**があるとします。

[^directives]: もちろん正確には#defineや#ifの展開などいろいろなことをします。

```c:header.h
int this_is_function_declaration(void);
```

```c:source.c
#include "header.h"
int main(void)
{
}
```

source.cは```#include "header.h"```でheader.hを読み込んでいますね。ところでこの#includeのように"#"から始まる命令を**プリプロセッサ・ディレクティブ**といいます。「ディレクティブ(directive)」は「指示」という意味で、ここでは「プリプロセッサに対する指示」という意味になります。

私たちは普段何気なくgccにソースコードを突っ込んでいますが、実はその内部では**プリプロセッサ**というものが呼ばれています。これは読んで字のごとく[^preprocessor]ソースコードの「前処理」をするものです。つまりソースコードが入力されてからの流れはざっくりとこんな流れになります。

```
ソースコードが入力される
↓
まずはプリプロセッサに渡して前処理
↓
満を持してコンパイル
```

[^preprocessor]: pre: 「事前に」，process: 「処理」，or: 「～するもの」

それではプリプロセッサはソースコードをどのように前処理するのか見てみましょう。**cpp**[^cpp]コマンドに先ほどのソースコードを突っ込むとこうなります。

[^cpp]: C PreProcessorの略。C++の略ではありません。

```bash
$ cpp -P source.c # -P オプションがないといろいろ余計なものがくっついてくる
int this_is_function_declaration(void);
int main(void)
{
}
```

```#include "header.h"```と書かれていた行に、代わりにheader.hの中身があらわれています。**これだけのことです**。やろうと思えばもっと無茶苦茶なこともできます。

```c:yabai_header.h
printf(
    "HELLO "
    "WORLD\n"
```

```c:yabai_source.c
int main(void)
{
#include "yabai_header.h"
    );
}
```

一見するとyabai_source.cはコンパイルできなさそうですが、プリプロセッサに通してみると特に問題なさそうなソースコードが出力されます。

```bash
$ cpp -P yabai_source.c
int main(void)
{
printf(
    "HELLO "
    "WORLD\n"
    );
}
```

そして実際コンパイルもできてしまいます。

```bash
$ gcc yabai_source.c # stdio.hをincludeしていないことによるちょっとした警告文は出てくる
$ ./a.out
HELLO WORLD
```

include文は怖くもなんともないということがお分かりいただけたでしょうか。

# 試されるinclude文くん

それではこちらをご覧ください。

``` c:include_itself.c
#include "include_itself.c"
```

これ、どうなるでしょうか。このソースコードを1回前処理するとこうなります。

```c
#include "include_itself.c"
```

・・・何も変わっていませんね。当然と言えば当然です。```#include "include_itself.c"```と書かれているところにinclude_itself.cの中身、つまり```#include "include_itself.c"```という文字列が置かれるのですから、結局何も変わりません。つまりこのソースコードの前処理は永遠に終わりません。

```bash
$ gcc include_itself.c
In file included from include_itself.c:1,
                 (以下略)
                 from include_itself.c:1,
                 from include_itself.c:1:
include_itself.c:1:28: error: #include nested too deeply
    1 | #include "include_itself.c"
      |                            ^
```

gccもエラーを吐いて終了します。

# 本題

それではこれらを踏まえたうえで、このソースコードはどうでしょう。

```c:include_itself_twice.c
#include "include_itself_twice.c"
#include "include_itself_twice.c"
```

**嫌な予感しかしない**ですね。

```bash
$ gcc include_itself_twice.c
In file included from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 from include_itself_twice.c:1,
                 (以下果てしなく続くエラーメッセージ)
```

このエラーメッセージは何行あるのでしょうか。それを見積もるために、「includeすべきファイルの探索を早めに打ち切る」ようにさせましょう。わりと新しめのgccには **-fmax-include-depth** オプションがあり、includeすべきファイルの探索の深さに制限を掛けられます。
ちょっとずつファイルの探索を深くしながらエラーメッセージの行数を数えていくと、

```bash
$ gcc -fmax-include-depth=1 include_itself_twice.c 2>&1 | wc -l
6
$ gcc -fmax-include-depth=2 include_itself_twice.c 2>&1 | wc -l
14
$ gcc -fmax-include-depth=3 include_itself_twice.c 2>&1 | wc -l
30
$ gcc -fmax-include-depth=4 include_itself_twice.c 2>&1 | wc -l
62
$ gcc -fmax-include-depth=5 include_itself_twice.c 2>&1 | wc -l
126
```

どうやら探索の深さを$N$と置くとエラーメッセージは$2^{N+2}-2$行ありそうです。そしてgccはデフォルトでは**深さ200**までincludeすべきファイルの探索を継続します。したがって先ほどのエラーメッセージは $2^{202}-2 \approx 6.43 \times 10^{60}$ 行、すなわち約**6那由多行**続くことになります。

私のマシンではエラーメッセージは10秒で約200万行[^note]出てきたので、このエラーメッセージが出終わるころには**1極年**経っています。**56億7千万年後**に弥勒菩薩が人類の救済にやってきても **0.000000000000000000000000000000000000556%** しか終わっていません。終わらない仕事を託されたコンピューターは我々人類を恨むでしょうか・・・

[^note]: エラーをファイルに書き出させた場合。端末に表示させた場合はもっと遅くなりそう。

# おまけ１

ネタ記事とはいえこれだけだとあんまりなので、include文を使った面白い小ネタをご紹介します。それがこちらです。

```c:include_tty.c
#include </dev/tty>
```

これをこうするとこうなります。

```bash
$ gcc include_tty.c
#include <stdio.h>
int main(void)
{
    puts("Hello World");
    return 0;
}
$ ./a.out
Hello World
```

**はっ？**

## 解説

Linuxには/dev/nullや/dev/randomなどあたかもファイルであるかのように扱える便利な**デバイスファイル**があるのですが、**/dev/tty**もその一つです。/dev/ttyは端末デバイスを表しています。つまり/dev/ttyを読み込むことで、端末から入力された文字列を読み込むことができます。

つまりこれ↓をgccに渡すと、**ターミナルから入力された文字列がinclude_tty.cに書いてあることになります**。

```c:include_tty.c
#include </dev/tty>
```

よってコンパイル時に任意のソースコードを記述できるようになります（Ctrl+Dで終了）。

```bash
$ gcc include_tty.c
#include <stdio.h>
int main(void)
{
    for (int i = 1; i <= 20; ++i) {
        if (i % 15 == 0) {
            printf("fizzbuzz\n");
        } else if (i % 5 == 0) {
            printf("buzz\n");
        } else if (i % 3 == 0) {
            printf("fizz\n");
        } else {
            printf("%d\n", i);
        }
    }
    return 0;
}
$ ./a.out
1
2
fizz
4
buzz
fizz
7
8
fizz
buzz
11
fizz
13
14
fizzbuzz
16
17
fizz
19
buzz
```

**EDIT:** コメント欄でご教授いただきました！**```#include</dev/tty>```すら要らなかった。**

```bash
$ gcc -xc -
#include <stdio.h>
int main(void)
{
    puts("Hello World");
    return 0;
}
$ ./a.out
Hello World
```

# おまけ2

include文に並んでよくつかわれるプリプロセッサ・ディレクティブに**define**文があります。define文はこのように使います。

```c:define_example.c
#include <stdio.h>
#define HELLO "Hello World"
int main(void)
{
    puts(HELLO);
}
```

```bash
$ gcc define_example.c
$ ./a.out
Hello World
```

つまり```#define A B```とすると、ソースコード中に出現するAがBで置き換えられることになります。

ところでdefineの具体的な内容はコンパイラオプションで決めることもできます。

```c:define_example2.c
#include <stdio.h>
int main(void)
{
    puts(HELLO);
}
```

```bash
$ gcc define_example2.c -DHELLO="\"Hello World\""
$ ./a.out
Hello World
```

これを使ってズルをすると**クワイン**が簡単に作れます。

**クワイン**とは、「自分自身のソースコードを出力するプログラム」のことです。作ったことがない方はぜひ挑戦してみてください。いい頭の体操になります。

そしてこのクワイン問題に対するズルい解がこちらです。

```c:zurui_quine.c
#include <stdio.h>
int main(void)
{
    puts(FOO);
    return 0;
}
```

```bash
$ gcc zurui_quine.c -DFOO="\"$(sed ':a;N;$!ba;s/\n/\\n/g' zurui_quine.c)\""
$ ./a.out
#include <stdio.h>
int main(void)
{
    puts(FOO);
    return 0;
}
```

詳しい説明は省略しますが、sedコマンドでなんかイイ感じにうまいことやります。それでFOOというシンボルにzurui_quine.cというソース文字列自体を持たせることで強引にクワインを実現しています。

# まとめ
オチらしきオチは特にありません。お付き合いくださりありがとうございました。
