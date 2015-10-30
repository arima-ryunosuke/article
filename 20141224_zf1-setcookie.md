# ZendFramework1 で setcookie する

ZF1 で `Set-Cookie` ヘッダを吐きたかったんですが、ググると `Zend_Http_Cookie` ばっかり引っかかります。
`Zend_Http_Cookie` は `Zend_Http_Client` で利用するもので、フロントコントローラでの利用は想定されていません(多分)。

さらにググると、ネイティブの `setcookie` 関数を使用している例も結構見つかります。
「まさかできないわけないだろー」と調べた所、無事`Zend_Http_Header_SetCookie` クラスを発見出来ました。
このクラスを使用するとクッキーの属性などをオブジェクティブに扱えるので便利です。

…が、実は公式ドキュメントに書いてあったりします。
[レスポンスオブジェクト - Zend_Controller - Zend Framework](http://framework.zend.com/manual/1.12/ja/zend.controller.response.html#zend.controller.response.headers.setcookie)

上記から引用：

```php
$cookie = new Zend_Http_Header_SetCookie();
$cookie->setName('foo')
       ->setValue('bar')
       ->setDomain('example.com')
       ->setPath('/')
       ->setHttponly(true);
$this->getResponse()->setRawHeader($cookie);
```

`Zend_Http_Header_SetCookie` でググっても 49 件しかヒットしないんだぜ？
結構ありそうな処理なのになぜ…。
