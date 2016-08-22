# なんか create_function が速い件

```php
<?php

// グローバルなただの関数(内容に意味はないです)
function gfunc($a)
{
    $b = $a + 51;
    $c = $b * 2;
    if ($c === 111) {
        return;
    }
    $s = 0;
    for ($i = 0; $i < 22; $i++) {
        $s += $a;
    }
    return $s;
}

// create_function した関数(内容に意味はないです)
$cfunc = create_function('$a', '
    $b = $a + 51;
    $c = $b * 2;
    if ($c === 111) {
        return;
    }
    $s = 0;
    for ($i = 0; $i < 22; $i++) {
        $s += $a;
    }
    return $s;
');

// 無名関数(内容に意味はないです)
$closure = function ($a) {
    $b = $a + 51;
    $c = $b * 2;
    if ($c === 111) {
        return;
    }
    $s = 0;
    for ($i = 0; $i < 22; $i++) {
        $s += $a;
    }
    return $s;
};

$t = microtime(true);
for ($i = 0; $i < 100000; $i++) gfunc($i);
echo 'gfunc    :', microtime(true) - $t, PHP_EOL;
// double(0.39156198501587)

$t = microtime(true);
for ($i = 0; $i < 100000; $i++) $cfunc($i);
echo '$cfunc   :', microtime(true) - $t, PHP_EOL;
// double(0.27994298934937) ← ！！！！

$t = microtime(true);
for ($i = 0; $i < 100000; $i++) $closure($i);
echo '$closure :', microtime(true) - $t, PHP_EOL;
// double(0.59706401824951)
```

どういうこっちゃい。

グローバルな関数より create_function の方が速いのは理由に全く検討がつかない。

関係ないけど、無名関数が遅めで可哀想なので、リフレクションでコード部分取って create_function すると無名関数を高速化できる？ と思ってやってみた。

```php
<?php

// 無名関数を高速化する関数（実装はかなり適当）
function fast_closure(\Closure $c)
{
    $ref = new \ReflectionFunction($c);
    $sl = $ref->getStartLine();
    $el = $ref->getEndLine();
    $ll = $el - $sl - 1;

    // コードスタイルとか use とか scopeClass とかで判定してダメそうならそのまま返す(多分無理)
    if (false) {
        return $c;
    }

    // 引数部分（create_function の第1引数）
    $params = $ref->getParameters();
    $args = array();
    foreach ($params as $param) {
        $arg = '$' . $param->getName();
        if ($param->isPassedByReference()) {
            $arg = '&' . $arg;
        }
        if ($param->isDefaultValueAvailable()) {
            $arg = $arg . '=' . var_export($param->getDefaultValue(), true);
        }
        $args[] = $arg;
    }

    // コード部分（create_function の第2引数）
    $lines = file($ref->getFileName());
    $lines = array_slice($lines, $sl, $ll);
    $code = ltrim(implode("\n", $lines), "\r\n\t {");

    // 無名関数のコードブロックで create_function して返す
    return create_function(implode(',', $args), $code);
}

$fclosure = fast_closure($closure);
$t = microtime(true);
for ($i = 0; $i < 100000; $i++) $fclosure($i);
echo '$fclosure:', microtime(true) - $t, PHP_EOL;
// double(0.28115391731262)
```

おおお、速くなった。超黒魔術。

ちなみに手持ちの

- php 5.4.16
- php 5.5.31
- php 5.6.16
- php 7.0.4

で全部同じだった（php7 はむしろ 2.5 倍くらい早かった）。

まぁ create_function なんて前世紀の遺物だしこんな些細な速度差は気にしたら負けだと思う。
