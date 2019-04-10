# ユニットテストで自動で clearstatcache する

## 何がしたいの？

下記のようなテストがあります。

```php:ExampleTest.php
<?php
class ExampleTest extends \PHPUnit\Framework\TestCase
{
    function test()
    {
        // mtime を 1234567890 に更新します
        touch(__FILE__, 1234567890);
        // その値になっているはずです
        $this->assertEquals(1234567890, filemtime(__FILE__));

        // mtime を 1234567899 に更新します
        touch(__FILE__, 1234567899);
        // その値になっているはずです
        $this->assertEquals(1234567899, filemtime(__FILE__));
    }
}
```

まぁ特に意味はないんですが、実際は何かのテストでファイルシステムをいじってると思ってください。
このテスト、一見通りそうですが、**通りません**。

```plain:出力
F                                                                   1 / 1 (100%)

Time: 138 ms, Memory: 6.00MB

There was 1 failure:

1) ExampleTest::test
Failed asserting that 1234567890 matches expected 1234567899.
```

原因は php が stat コールをキャッシュするからで、こういったテストには `clearstatcache` を呼び出す必要があります。
（分かってはいるんだけど毎回忘れる）。

```php:ExampleTest.php
<?php
class ExampleTest extends \PHPUnit\Framework\TestCase
{
    function test()
    {
        // mtime を 1234567890 に更新します
        touch(__FILE__, 1234567890);
        clearstatcache();
        // その値になっているはずです
        $this->assertEquals(1234567890, filemtime(__FILE__));

        // mtime を 1234567899 に更新します
        touch(__FILE__, 1234567899);
        clearstatcache();
        // その値になっているはずです
        $this->assertEquals(1234567899, filemtime(__FILE__));
    }
}
```

なんとなく不格好ですし、やはり忘れやすいです。
今回はこれをなんとかしようというお話です。
（まぁテストメソッドを分ければ [PHPUnit がクリアしてくれる](https://github.com/sebastianbergmann/phpunit/blob/8.1.0/src/Framework/TestCase.php#L829)んですがね…）。

## 実際にとった方法

stat キャッシュを無効にする方法や設定はどうやら無いようです。
ただ、 php には [ticks](https://www.php.net/manual/ja/control-structures.declare.php#control-structures.declare.ticks) という各ステートメントごとに処理を挟む機能があるのでそれを使用します。

PHPUnit の bootstrap.php あたりに下記を追加します。

```php:bootstrap.php
register_tick_function(function () {
    clearstatcache();
});
```

これは tick 関数を追加するだけで、実際には declare 宣言が必要なので Test ファイルの冒頭に下記を追加します。

```php:ExampleTest.php
<?php
declare(ticks=1);

class ExampleTest extends \PHPUnit\Framework\TestCase ...
```

これだけでテストが通るようになります。

```plain:出力
.                                                                   1 / 1 (100%)

Time: 107 ms, Memory: 6.00MB

OK (1 test, 2 assertions)
```

ただ、所感ですが毎ステートメントで登録された処理が呼ばれるので、なんとなくテスト時間が増大しそうです（**未計測**）。
ファイルシステム関係以外のテストでは `clearstatcache` を実行する必要はないので省きたいです。

まぁファイル単位でも良いんですが、 declare 宣言はブロックで局所的に当てることができるので下記のようにもできます。

```php:ExampleTest.php
<?php
class ExampleTest extends \PHPUnit\Framework\TestCase
{
    function test()
    {
        declare(ticks=1){
            // mtime を 1234567890 に更新します
            touch(__FILE__, 1234567890);
            // その値になっているはずです
            $this->assertEquals(1234567890, filemtime(__FILE__));

            // mtime を 1234567899 に更新します
            touch(__FILE__, 1234567899);
            // その値になっているはずです
            $this->assertEquals(1234567899, filemtime(__FILE__));
        }
    }
}
```

`register_tick_function` は処理を登録するだけで、実際に declare しなければ何も動作に影響は与えないはずです。
これで本当に「局所的に自動 clearstatcache」が実現できます。
