# $http_response_header にはリダイレクトが全部入ってくる

使い捨てスクリプトで (http で) `file_get_contents` + `$http_response_header` は手軽なのでよく使うんですが、`$http_response_header` が大量にあることがあって、なんだろうと調べてみたらリダイレクトされたときのヘッダーが全部格納されてました。

```php:aaa.php
<?php
header('Location: /bbb.php');
```

```php:bbb.php
<?php
header('Location: /ccc.php');
```

```php:ccc.php
<?php
header('Location: /zzz.php');
```

```php:zzz.php
<?php
echo 'zzz!';
```

このような4つのファイルを作って `php -S localhost:8000` して built in server で試してみます。

```php
<?php
echo file_get_contents('http://localhost:8000/aaa.php');
print_r($http_response_header);
```

上記の出力は下記のようになります。

```
zzz!
Array
(
    [0] => HTTP/1.0 302 Found
    [1] => Host: localhost:8000
    [2] => Connection: close
    [3] => X-Powered-By: PHP/7.0.27
    [4] => Location: /bbb.php
    [5] => Content-type: text/html; charset=UTF-8
    [6] => HTTP/1.0 302 Found
    [7] => Host: localhost:8000
    [8] => Connection: close
    [9] => X-Powered-By: PHP/7.0.27
    [10] => Location: /ccc.php
    [11] => Content-type: text/html; charset=UTF-8
    [12] => HTTP/1.0 302 Found
    [13] => Host: localhost:8000
    [14] => Connection: close
    [15] => X-Powered-By: PHP/7.0.27
    [16] => Location: /zzz.php
    [17] => Content-type: text/html; charset=UTF-8
    [18] => HTTP/1.0 200 OK
    [19] => Host: localhost:8000
    [20] => Connection: close
    [21] => X-Powered-By: PHP/7.0.27
    [22] => Content-type: text/html; charset=UTF-8
)
```

これ多分単純に突っ込んでるだけですよね。
つまりよくある

```php
preg_match('/HTTP\/1\.[0|1|x] ([0-9]{3})/', $http_response_header[0], $matches);
$statusCode = (int) $matches[1];
if ($statusCode === 200) {
    // 
}
```

このような処理はせっかくリダイレクトの森を抜けてボディが得られたのに 200 以外と判定されて失敗することになります。
しっかりやるなら「最後のヘッダのステータスコード」を取得する必要があるけど使い捨てでそこまでするのはちょっと…。

まぁ知っておくといつか得するかもしれない。
