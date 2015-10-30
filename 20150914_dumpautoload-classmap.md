# composer dumpautoload -o は全クラスマップを生成する

`-o` は `--optimize` の意味です。
公式マニュアルにも書かれている既知の機能なのですが、ダダハマりしたのでメモっておきます。

結論から述べると `-o` を付けると「オートロードされるであろうファイルのクラスマップ」を生成するのではなく、「指定ディレクトリ以下のクラスマップ」を生成します。


## ディレクトリ構成

下記です。

```bash:tree
/path/to/project
├── composer.json
├── main.php
└── src
    └── package
        └── sab
            └── Test.php
```

確認用の composer.json です。

```json:composer.json
{
	"autoload" : {
		"psr-4" : {
			"vendor\\package\\" : "src/package"
		}
	}
}
```

実行用の php です。

```php:main.php
<?php
require __DIR__ . '/vendor/autoload.php';

var_dump(new \vendor\package\sub\Test());
```

オートロードされる Test クラスです。

```php:src/package/sab/Test.php
<?php
namespace vendor\package\sub;

class Test {}
```


## -o で composer install する

`composer install -o` します。

```bash
$ composer install -o
Loading composer repositories with package information
Installing dependencies (including require-dev)
Nothing to install or update
Generating optimized autoload files
```

4行目で「Generating optimized autoload files」となっており、`-o` が効いています。
そして確認用 php を実行します。

```bash
$ php main.php
class vendor\package\sub\Test#2 (0) {
}
```

きちんとオートロードできていることが確認できます。


## -o なしで composer install する

`composer install` します。

```bash
$ composer install
Loading composer repositories with package information
Installing dependencies (including require-dev)
Nothing to install or update
Generating autoload files
```

4行目で「Generating autoload files」となっており、`-o` なしの install が実行されています。
そして確認用 php を実行します。

```bash
$ php main.php
Fatal error: Class 'vendor\package\sub\Test' not found in /path/to/project/main.php on line 5

Call Stack:
    0.0001     223328   1. {main}() /path/to/project/main.php:0
```

オートロードできませんでした。

## なんで？

気づいている人もいると思いますが、psr 的に「Test.php」が配置されるであろうパスが間違っています。

正：`src/package/sub/Test.php`
誤：`src/package/sab/Test.php`

`-o` で dumpautoload すると `vendor/composer/autoload_classmap.php` にクラスマップが生成されるので覗いてみます。

```php:vendor/composer/autoload_classmap.php
return array(
    'vendor\\package\\sub\\Test' => $baseDir . '/src/package/sab/Test.php',
);
```

sub と sab に注目です。
間違った？パスでクラスマップが生成されています。

どうも `-o` があると、指定ディレクトリ（この場合 composer.json の「autoload.psr-4」のディレクトリ）以下を単に総なめしてクラスマッピングするだけのようです。
いわゆる psr 的な名前空間とパスの対応は見てくれません。

この例だとまぁ分かりやすいですが、実際にハマった例だと

正：UnitTest
誤：UniTest

でした。数週間の間、「なんか最近 `-o` が無いとオートロードが効かないなぁ」と？？？な心境になっていました。

ただ `-o` するにしても、普通は「本番環境で `-o` 、開発環境では `-o` なし」のような運用になると思うので、そこまで致命的な事にはならないと思います。
