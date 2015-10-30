# php でサクッとベンチマークを取る関数

下記のような関数を作りました。

```php:benchmark.php
function benchmark($callback)
{
    $PERIOD = 1;
    $FORMAT = "%-24s %9s called.%s";
    
    if (is_callable($callback, false, $callname))
    {
        if ($callback instanceof Closure)
        {
            $ref = new ReflectionFunction($callback);
            $callname = $ref->getFileName() . '#' . $ref->getStartLine();
        }
    }
    else
    {
        $callname = $callback;
        $callback = create_function("", "$callback;");
    }
    
    $args = array_slice(func_get_args(), 1);
    
    $end = microtime(true) + $PERIOD;
    for ($count = 0; microtime(true) <= $end; $count ++)
    {
        call_user_func_array($callback, $args);
    }
    
    printf($FORMAT, $callname, number_format($count), PHP_EOL);
}
```

callable な引数と、それを呼ぶときの引数を与えると、1 秒間で何回呼べたかを表示します。
（実行可能なコード文字列を与えても OK なようになってますが、これは自分用のおまけです。どうせベンチマークだし）。

```php:sample.php
require_once 'benchmark.php';

class C
{
    public function __get($name)
    {
        if (method_exists($this, $name))
        {
            return array($this, $name);
        }
    }

    public function f()
    {
        return "my name is " . get_class() . ".";
    }

    public function g()
    {
        $name = get_class();
        return "my name is {$name}.";
    }
}

benchmark("mt_rand", 1, 100);
benchmark("rand", 1, 100);

benchmark('(float)"1.2"');
benchmark('floatval("1.2")');

benchmark(function ($v) {return $v * $v;}, 4);
benchmark(function ($v) {return pow($v, 2);}, 4);

$c = new C();
benchmark(array($c, 'f'));
benchmark(array($c, 'g'));
benchmark($c->f);
benchmark($c->g);
```

上記の実行結果は下記です。

```
mt_rand                    879,020 called.
rand                       847,397 called.
(float)"1.2"               716,748 called.
floatval("1.2")            591,736 called.
/path/to/sample.php#32     714,772 called.
/path/to/sample.php#33     603,412 called.
C::f                       664,103 called.
C::g                       616,159 called.
C::f                       651,382 called.
C::g                       606,337 called.
```

関数はそのまま関数名を、eval な文字列はそのままそれを、クロージャの場合は定義場所を、メソッドの場合は「クラス::メソッド」を、それぞれ名前として表示します。

この結果により、

* rand 関数より mt_rand 関数の方が速い（何度か実行すると逆転したので誤差レベル）
* floatval 関数より float キャストの方が速い
* pow 関数で 2 乗するより単純に掛け算したほうが速い
* 文字列埋め込みより文字列結合の方が速い

的なことが推測できます。

個人的に「こうしたほうが速いかなー、いや、こうかなー」のような試行錯誤を結構するので、こういう関数は有用だったりします。

※ クラスC の `__get` はメソッドを呼び出し可能な形式で返すためのものです。本筋とは全く関係ありません
※ 個人的に `array($c, 'f')` より `$c->f` で callable にしたいんです…（メソッド名を文字列で指定するのがなんか嫌）
