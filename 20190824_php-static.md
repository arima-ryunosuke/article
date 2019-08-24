# static 変数の初期値に注意

php でコードを書いていて「クラス/インスタンス間の static 変数ってどういう挙動だっけ？」と思って調べていたところ [PHPの静的変数 (static変数) の挙動まとめ](https://qiita.com/trashtoy/items/f4e2a97765e620bb2828)を見つけて、「あぁ、なるほど」と納得したんですが、微妙におかしな挙動を示したので備忘として残しておきます。

- OS: CentOS7
- php: 7.2.14

結論を言うと、上述の記事の通り確かに static 変数はクラス間で独立した挙動を示しますが、その**初期値が状況によって異なります**。

## 再現コード

上述の記事から引用して乗っかります。

> では、Test1 を継承した子クラスから実行した場合はどうなるでしょうか。結果は以下のコードから。(PHP 5.1.0 - 5.5.13 で確認)

```php
<?php
class Test1
{
    public static function testStaticMethod()
    {
        static $num = 0;
        $num++;
        echo __METHOD__ . $num . PHP_EOL;
    }
}

class Test2 extends Test1
{
}

class Test3 extends Test1
{
}

Test1::testStaticMethod();
Test2::testStaticMethod();
Test3::testStaticMethod();
Test1::testStaticMethod();
Test2::testStaticMethod();
Test3::testStaticMethod();
```

> このコードは以下を出力します。

```php
Test1::testStaticMethod1
Test1::testStaticMethod1
Test1::testStaticMethod1
Test1::testStaticMethod2
Test1::testStaticMethod2
Test1::testStaticMethod2
```

> この出力内容から分かる通り、Test1, Test2, Test3 のそれぞれのクラスで $num が独立して管理されていることが分かります。

この動作に全くおかしなところはありません。

ただし、クラスがロードされていないと static 変数の初期値が「既に設定されている親クラスの static 変数」に設定されてしまうようです。

```php
<?php
class Test1
{
    public static function testStaticMethod()
    {
        static $num = 0;
        $num++;
        echo __METHOD__ . $num . PHP_EOL;
    }
}

Test1::testStaticMethod();

if (true) {
    class Test2 extends Test1
    {
    }
}
Test2::testStaticMethod();
Test2::testStaticMethod();

if (true) {
    class Test3 extends Test1
    {
    }
}
Test3::testStaticMethod();
Test3::testStaticMethod();
```

php は一旦ソース全体をなめてシンボルの解決を行うので、それを防ぐためにクラス定義を `if (true)` で包んでいます。
要するにクラスのオートロードと同じ挙動になるような模倣です（≒必要になったときに読み込む）。

このコードは以下を出力します。

```
Test1::testStaticMethod1
Test1::testStaticMethod2
Test1::testStaticMethod3
Test1::testStaticMethod2
Test1::testStaticMethod3
```

`Test2::testStaticMethod()` は初回呼び出しにも関わらず、値が `2` になっています。
`Test3::testStaticMethod()` も同様です。
その後の呼び出しは独立して管理されているので、それぞれで `3` になります。

これは Test1 の既に代入されている `static $num` に影響を受けているためだと思われます。
実際、 `Test1::testStaticMethod()` を呼ばなければ `Test2` も `Test3` もそれぞれ `1` から始まります。

一応実行結果を置いておきます。

- https://3v4l.org/C79UV
- https://3v4l.org/trNoO

## 所感

公式ドキュメントに「クラス間の static 変数の挙動」や「継承した場合の static 変数の初期値」などの記載を見つけることは出来なかったんですが、おそらくこれはクラスローディングの際の php 不具合のような気がします。
（ロード時に static 部分が共用されてしまっている？）。

不具合ならば修正されるでしょうが、現在のところ「クラスがロードされているか？」で挙動が変わってしまうので、下手すると深刻・非常に分かりにくいバグの原因になります。
「クラスがロードされているか？」は状況によってかなり異なるためです。
とりあえず頭の片隅にでも入れておくと良いかもしれません。
