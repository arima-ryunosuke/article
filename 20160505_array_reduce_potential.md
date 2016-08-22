# array_reduce を使い倒す

## やりたいこと

配列の中から

- 小文字ではじまる要素を抽出して
- 先頭に "!" プレフィックスを付けたい


## array_filter + array_map

```php
function f1($array)
{
    return array_map(function ($item) {
        return '!' . $item;
    }, array_filter($array, function ($item) {
        return ctype_lower($item[0]);
    }));
}
var_dump(f1(['Aa', 'bb', 'cC', 'DD']));
/*
array(2) {
  [1] =>
  string(3) "!bb"
  [2] =>
  string(3) "!cC"
}
*/
```

array_filter + array_map で実装したものです。
おそらく真っ先に頭に浮かぶ実装だと思いますし、何も間違っていないと思います。
ただ、php の array_*** 周りの引数は順番に統一性がなく、無名関数が出てくると途端に視認性が悪くなります。
例えば私は上の関数はパッと見「map してから filter する」ようにしか見えません。まず処理部分に視線が奪われるからです。

```php
function f1_($array)
{
    return array_map(array_filter($array, function ($item) {
        return ctype_lower($item[0]);
    }, function ($item) {
        return '!' . $item;
    }));
}
```

こう書ければいいのに array_map が空気を読んでいないんです（多分可変引数のためなんだろうけど）。
まぁそもそもワンライナーで書かなきゃ済む話なんですけどね。


## array_reduce

```php
function f2($array)
{
    return array_reduce($array, function ($carry, $item) {
        if (ctype_lower($item[0])) {
            $carry[] = '!' . $item;
        }
        return $carry;
    }, []);
}
var_dump(f2(['Aa', 'bb', 'cC', 'DD']));
/*
array(2) {
  [0] =>
  string(3) "!bb"
  [1] =>
  string(3) "!cC"
}
*/
```

array_reduce で実装したものです。
array_reduce はその関数名から配列→スカラー値にする関数に感じてしまいますが、別にそんなことはなく「今までの結果が引数で渡ってくる array_walk」のように捉えることが出来ます。
filter + map が1つの関数に集約され、個人的にはとても読みやすいです。

余談ですが、array_reduce 版だとキーが死んで数値連番になります。
が、こういった処理の場合、たいてい数値連番を期待していて、後で array_values をカマしたりするのでむしろ都合がいいくらいです。

ちなみにキーが死んで困る場合は array_keys + use でこうするのが好みです。

```php
function f2_($array)
{
    return array_reduce(array_keys($array), function ($carry, $item) use ($array) {
        if (ctype_lower($array[$item][0])) {
            $carry[$item] = '!' . $array[$item];
        }
        return $carry;
    }, []);
}
var_dump(f2_(['Aa', 'bb', 'cC', 'DD']));
/*
array(2) {
  [1] =>
  string(3) "!bb"
  [2] =>
  string(3) "!cC"
}
*/
```


## foreach

```php
function f3($array)
{
    $result = [];
    foreach ($array as $key => $item) {
        if (ctype_lower($item[0])) {
            $result[$key] = '!' . $item;
        }
    }
    return $result;
}
var_dump(f3(['Aa', 'bb', 'cC', 'DD']));
/*
array(2) {
  [1] =>
  string(3) "!bb"
  [2] =>
  string(3) "!cC"
}
*/
```

foreach で実装したものです。
ぶっちゃけ一番わかりや（ｒｙ

## まとめ

個人的に `array_reduce` は結構好きな関数で、もっと使われて欲しいんですが、どうも

- 不必要な参照、use してたり
- array_sum, array_product などで十分だったり

なサンプルが多く、あんまりしっかり使われることがない印象です。
ので紹介も兼ねてこんな記事を書いてみました。

特に filter + map って相性が抜群で、非常にしばしば出てくる処理なので array_reduce で書き換えたくなったりします。

なお「文字列の1文字目を `[0]` でアクセス」してるのは単にサボってるだけです。
本題ではないのであまり突っ込まないでください。
