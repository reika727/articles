---
title: GNU C ライブラリのジョーク（？）関数 "memfrob"
tags:
  - C
  - ネタ
  - GNU
private: false
updated_at: '2023-03-14T02:11:44+09:00'
id: 0b7c8a4257ff78855b21
organization_url_name: null
slide: false
ignorePublish: false
---
GNU C ライブラリに [memfrob](https://www.gnu.org/software/libc/manual/html_node/Obfuscating-Data.html) という変わった関数があったのでご紹介します。

`memfrob(void *mem, size_t length)` は `mem` からはじまる `length` バイトの領域に対し、1 バイトごとに [42](https://www.google.com/search?q=人生、宇宙、全ての答え) との排他的論理和を計算して上書きします。

つまりこういうことです。

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>

int main(void)
{
    char s[3] = { 9, 12, 5 };

    /* s を表示 */
    for (int i = 0; i < 3; ++i) printf("%d ", s[i]); printf("\n");

    /* 書き換えて表示 */
    memfrob(s, 3);
    for (int i = 0; i < 3; ++i) printf("%d ", s[i]); printf("\n");

    /* もう一回やる */
    memfrob(s, 3);
    for (int i = 0; i < 3; ++i) printf("%d ", s[i]); printf("\n");
}

```

```
9 12 5
35 38 47
9 12 5
```

排他的論理和を取ってるだけなので、二回やると元に戻るのがわかります。

:::note info
[Wiktionary によると](https://en.wiktionary.org/wiki/frob)、"frob" という動詞（あまり一般的ではない）は「ぐちゃぐちゃにする」、「いじくりまわす」という意味で使われるようです。

またシリコンバレーでは「自分にはわかるけど人に説明するのは難しい（あるいはめんどくさい）ことをやる」という意味のスラングとしても用いられるようです。

（Wiktionary から引用）Why don't you go get lunch? I need to **frob** with this thing for about 30 minutes and then we'll be good to go.
（拙訳）昼飯食べてていいぞ。こっちはあと三十分くらいこれを**アレしなきゃ**いけないから。
:::

この関数で文字列のちょっとした暗号化もできます。

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>

int main(void)
{
    /* ASCII 印字可能文字に変換されるのは大文字・小文字アルファベットと @ [ \ ] ^ _ ` { | } ~ のみ */
    /* 空白文字は改行に変換されてしまうのでちょっと扱いにくい */
    char s[] = "HelloWorld";
    puts(s);
    memfrob(s, strlen(s));
    puts(s);
    memfrob(s, strlen(s));
    puts(s);
}
```

```
HelloWorld
bOFFE}EXFN
HelloWorld
```

**言うまでもないことですが**、実用性は一切ない関数です。なんでこんなもん作ったんですかね・・・
