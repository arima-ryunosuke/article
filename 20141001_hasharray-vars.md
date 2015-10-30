# 変数から連想配列を作る

## 前提

下記のようなことをサクッとやりたいとします。

```php
$hoge = 1;
$fuga = 2;
$piyo = 3;
$array = array(
   'hoge' => $hoge,
   'fuga' => $fuga,
   'piyo' => $piyo,
);
```

これ、書くたびに思うんですよ、冗長だなぁと。
まぁこれだけなら `compact` という標準の関数で一発で出来ますが、文字列として指定するのでシンタックスハイライトが効かなかったり、IDE に「未使用変数があるぞ」って怒られたりします。

```php:compact
$hoge = 1;
$fuga = 2;
$piyo = 3;
$array = compact('hoge', 'fuga', 'piyo');
```

※ さすがに Qiita でもシンタックスハイライトされないですね

要するに上記を

```php:理想
$hoge = 1;
$fuga = 2;
$piyo = 3;
$array = compact2($hoge, $fuga, $piyo);
```

このように書きたいわけです。

## どうすればできるか

冷静に考えれば分かりますが、おそらく**まっとうな方法では不可能**です。
`compact2` 内のコンテキストで、呼び出し元の「hoge」「fuga」「piyo」という変数名を得る手段が存在しないからです。
ので `debug_backtrace` を使用します。

`debug_backtrace` を使うと呼び出し元のファイル名と実行行が得られるので、適当にパースすれば変数名が得られます。

```php:compact2
function compact2()
{
    // 必要な変数群
    $trace = debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 1)[0];
    $num = func_num_args();
    $args = func_get_args();

    // 呼ぶ必要がない
    if ($num === 0)
    {
        throw new \InvalidArgumentException('arguments are empty.');
    }

    // 呼び出し元の1行を取得
    $content = file_get_contents($trace['file']);
    $lines = preg_split('/\R/u', $content, $trace['line'] + 1);
    $target = $lines[$trace['line'] - 1];

    // 1行内で複数呼んでいる場合のための配列
    $caller = array();
    $callers = array();

    // php パーシング
    $starting = false;
    $tokens = token_get_all('<?php ' . $target);
    foreach ($tokens as $token)
    {
        // トークン配列の場合
        if (is_array($token))
        {
            // 自身の呼び出しが始まった
            if (!$starting && $token[0] === T_STRING && $token[1] === $trace['function'])
            {
                $starting = true;
            }
            // 呼び出し中でかつ変数トークンなら変数名を確保
            else if ($starting && $token[0] === T_VARIABLE)
            {
                $caller[] = ltrim($token[1], '$');
            }
            // 上記以外の呼び出し中のトークンは空白しか許されない
            else if ($starting && $token[0] !== T_WHITESPACE)
            {
                throw new \UnexpectedValueException('argument allows variable only.');
            }
        }
        // 1文字単位の文字列の場合
        else
        {
            // 自身の呼び出しが終わった
            if ($starting && $token === ')')
            {
                $callers[] = $caller;
                $caller = array();
                $starting = false;
            }
        }
    }

    // 同じ引数の数の呼び出しは区別することが出来ない
    $length = count($callers);
    for ($i = 0; $i < $length; $i++)
    {
        for ($j = $i + 1; $j < $length; $j++)
        {
            if (count($callers[$i]) === count($callers[$j]))
            {
                throw new \UnexpectedValueException('argument is ambiguous.');
            }
        }
    }

    // 引数の数が一致する呼び出しを返す
    foreach ($callers as $caller)
    {
        if (count($caller) === $num)
        {
            return array_combine($caller, $args);
        }
    }
}
```

以下は簡単なテストと動作確認です。
既存コードから継ぎ接ぎしたので `setExpectedException` とかは気にしないでください。 

```php:test
// 普通に呼ぶ
$this->assertEquals(compact('hoge'), compact2($hoge));
$this->assertEquals(compact('piyo', 'fuga'), compact2($piyo, $fuga));

// 同一行で2回呼んでも引数の数が異なれば区別できる
$this->assertEquals(array(compact('hoge'), compact('piyo', 'fuga')), array(compact2($hoge), compact2($piyo, $fuga)));

// 引数の数が同じでも行が異なれば区別できる
$this->assertEquals(array(compact('hoge'), compact('fuga')), array(
    compact2($hoge),
    compact2($fuga),
));

// 引数なしは呼ぶ意味が無い
$this->setExpectedException('InvalidArgumentException', 'empty');
compact2();

// 即値は使用できない
$this->setExpectedException('UnexpectedValueException', 'variable');
compact2($hoge, 1);

// 同一行に同じ引数2つだと区別出来ない
$this->setExpectedException('UnexpectedValueException', 'ambiguous');
$dummy = array(compact2($hoge), compact2($fuga));
```

正直言って、`compact` の100倍くらい遅かったり、どんな呼び出し方でも取得できるか分からないとか(クロージャとか使ったらどうなるんだろう)、完全にお遊びです。

あと PhpStorm だとシンタックスハイライトはともかく、「使用している変数」とみなしてくれるようです。
実用的にはそれで十分だと思います。
