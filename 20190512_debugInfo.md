# __debugInfo の隠れ仕様的なもの

php 5.6 より `__debugInfo` で `var_dump` の出力を弄れる様になってますが、その挙動についてドキュメントで触れられていないことがあるので少々補完します。
※ アンドキュメントってことは正しい仕様ではなく「現在たまたまそうなっているだけ」の可能性があります
※ 確認は全て手元の php 7.1.14 で行いました

## おさらい

```php
class Debug
{
    private   $privateField   = 1;
    protected $protectedField = 2;
    public    $publicField    = 3;
}

$debug = new Debug();
$debug->dynamicField = 4;

var_dump($debug);
```

このようなコードを実行すると下記のような出力が得られます。

```
class Debug#2 (4) {
  private $privateField =>
  int(1)
  protected $protectedField =>
  int(2)
  public $publicField =>
  int(3)
  public $dynamicField =>
  int(4)
}
```

このクラスに[マニュアル通り](https://www.php.net/manual/ja/language.oop5.magic.php#object.debuginfo)に `__debugInfo` を実装します。

```php
class Debug
{
    private   $privateField   = 1;
    protected $protectedField = 2;
    public    $publicField    = 3;

    public function __debugInfo()
    {
        return [
            'propSquared' => $this->privateField ** 2,
        ];
    }
}

$debug = new Debug();
$debug->dynamicField = 4;

var_dump($debug);
```

出力は下記のようになります。

```
class Debug#2 (1) {
  public $propSquared =>
  int(1)
}
```

まぁ仕様どおりですね。

### 隠れた仕様1（アクセスレベル）

実際、上記のような完全カスタムの用途が多いと思いますが、私が今回やりたかったのは「でかいプロパティを伏せる」でした。
巨大フィールドを抱えていると `var_dump` の出力がでかすぎて視認性が悪いからです。
これはその過程で気づいた仕様です（private とかを維持したままカスタムする方法がわからなかった）。

ドキュメントに一切記載がないのですが、どうも `__debugInfo` の返り値は[オブジェクトを配列キャストしたもの](https://www.php.net/manual/ja/language.types.array.php#language.types.array.casting)であるべきのようです。

- https://github.com/php/php-src/blob/php-7.1.14/ext/standard/var.c#L63-L77
- https://github.com/php/php-src/blob/php-7.1.14/Zend/zend_compile.c#L1327

public フィールドを配列キャストしても普通のキーになるため、素の配列を返すと public と判定されます。上記で `public $propSquared` となっているのはそのためです。
つまり、それらしい配列を返してやれば `__debugInfo` で private/protected（っぽい）出力をすることができます。

```php
    public function __debugInfo()
    {
        return [
            "\0Debug\0dummy1" => 'this is private',
            "\0*\0dummy2"     => 'this is protected',
        ];
    }
```

この出力は下記のようになり、あたかも dummy1 という private フィールドと dummy2という protected フィールドがあるかのような出力になります。

```
class Debug#2 (2) {
  private $dummy1 =>
  string(15) "this is private"
  protected $dummy2 =>
  string(17) "this is protected"
}
```

ここで冒頭に戻り、「でかい（特定）プロパティを伏せる」ためには配列キャストしてからそれらを伏せれば良いことが分かります。

```php
    public function __debugInfo()
    {
        $properties = (array) $this;
        unset($properties["\0Debug\0privateField"]);
        return $properties;
    }
```

```
class Debug#2 (3) {
  protected $protectedField =>
  int(2)
  public $publicField =>
  int(3)
  public $dynamicField =>
  int(4)
}
```

それらしいキーを伏せているため、 `$privateField` が消えています。

### 隠れた仕様2（再帰参照）

再帰参照を含むオブジェクトで `__debugInfo` を実装すると出力がやべぇことになります。

```php
class Debug
{
    public function __debugInfo()
    {
        return (array) $this;
    }
}

$debug = new Debug();
$debug->dynamicField = $debug;

var_dump($debug);
```

上記の出力は下記のようになります（長すぎるので抜粋）。

```
class Debug#2 (1) {
  public $dynamicField =>
  class Debug#2 (1) {
    public $dynamicField =>
    class Debug#2 (1) {
      public $dynamicField =>
      class Debug#2 (1) {
        public $dynamicField =>
                                      class Debug#2 (1) {
                                        ...
                                      }
      }
    }
  }
}
```

上記のインデントのズレは私のミスでもなんでもなく、本当に大量にネストされて遥か右方へ追いやられた結果です（前後は省略）。
なお、 `__debugInfo` を実装しなければこのようなことは起きません。

多分組み込みなら再帰の検出ができるけど、 `__debugInfo` を自前実装した結果、（配列なので）再帰の検出ができなくなってるのかな？（だとしても最終的に `...` が出るのが謎い）。
`__debugInfo` を実装するときは再帰に注意するか、明示的な（シンプルな）配列を返したほうが良さそうです。

### 隠れた仕様3（print_r）

これもドキュメントにないのですが、 `__debugInfo` を実装すると `print_r` にも影響が出ます（https://github.com/php/php-src/blob/php-7.1.14/Zend/zend.c#L186）。

```php
class Debug
{
    public function __debugInfo()
    {
        return ['field' => '__debugInfo の返り値です'];
    }
}

print_r($debug = new Debug());
```

実行すると下記のような出力になり、 `print_r` にも影響が出ていることが分かります。

```
Debug Object
(
    [field] => __debugInfo の返り値です
)
```

