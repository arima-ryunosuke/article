# class_exists でのオートロード

`class_exists` で定義済みクラスを探すときはマニュアルの通り、大文字小文字は区別されませんが（そもそも php の言語仕様としてクラス名の大文字小文字は区別しないはず）、定義されていないクラスを探そうとして、オートロードが走ると、ファイルシステムによってはうまくロードされないことがあります。

例えば下記のような構造で

```
hoge/
├── SampleClass.php
└── main.php
```

```php:hoge/SampleClass.php
<?php

Class SampleClass{}
```

```php:hoge/main.php
<?php

// 単純なオートロードを登録
spl_autoload_register(function ($classname)
{
    $filename = __DIR__ . DIRECTORY_SEPARATOR . $classname . '.php';
    if (file_exists($filename))
    {
        require_once $filename;
    }
});

// 小文字クラス名で class_exists
var_dump(class_exists('sampleclass'));
// 正式なクラス名で class_exists
var_dump(class_exists('SampleClass'));
// もう1回小文字クラス名で class_exists
var_dump(class_exists('sampleclass'));
```

上記を Windows 環境で実行すると下記のようになります。

```
bool(true)
bool(true)
bool(true)
```

Linux 環境で実行すると下記のようになります。

```
bool(false)
bool(true)
bool(true)
```

これは Windows(のファイルシステム) が大文字小文字のファイル名を同一視するからです。  
Windows 環境では最初の `class_exists` で問題なくオートロードされ、以後 `SampleClass` は定義済みクラスとなるので下の2つも `true` になります。  
Linux 環境では小文字のファイル名が存在しないため、1つ目ではロードされず、2つ目の正式クラス名でロードされ、その時点で定義済みとなるため、3つ目は `true` になります。

オートローダの実装にもよりますが、「ファイルが存在するなら読み込む」という動作のオートローダだとエラーすら出てくれません。
たしか composer はそんな感じの動作だったかと。

そもそも普通に考えて `class_exists` に起因した現象などではなく、単純にオートロードに起因しているだけなんですが、
「`class_exists` はオートロードしてくれるから安心だぜぇ」って思考だとハマるかもしれないです。
