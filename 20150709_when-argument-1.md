# $.when に引数を1つだけ渡すときの注意点

タイトル通り、$.when に引数を1つだけ渡すと結果の引数の構造が異なります。
version 1.11.1 です。

## 1つだけ渡す

下記のスクリプトで試します。

```javascript
// 引数2つで $.when ----- A
$.when(
	$.get('/hoge.json'),
	$.get('/fuga.json')
).done(function() {
	console.log(arguments);
});

// 引数1つで $.when ----- B
$.when(
	$.get('/hoge.json')
).done(function() {
	console.log(arguments);
});
```

A はいわゆる普通の使い方、B は普通はあんまりしないであろう引数1つの使い方です。

見づらくて申し訳ないですが、下記が実行結果です。


```json:Aの出力
{
  "0": [
    {
      "data": "hoge"
    },
    "success",
    {
      "readyState": 4
      /*長いので略*/
    }
  ],
  "1": [
    {
      "data": "fuga"
    },
    "success",
    {
      "readyState": 4
      /*長いので略*/
    }
  ]
}
```

```json:Bの出力
{
  "0": {
    "data": "hoge"
  },
  "1": "success",
  "2": {
      "readyState": 4
      /*長いので略*/
  }
}
```

A の引数にはいわゆる success の引数 `(Anything data, String textStatus, jqXHR jqXHR)` の**配列**が渡ってきています。これは想定通りです。
しかし、B の引数には上記 success の引数が**配列ではなく単値**で渡ってきています。

引数が1つか否かで結果の構造が異なってしまっています。
ここは要素1の配列で渡ってきて欲しいです。
これのせいで $.when の引数に配列を渡せるように改造していたんですが、ダダハマりしました…。

余談ですが、引数が可変でこそ $.when が生きると思うんですよね。
なんで配列渡しに対応してないんですかね。

## ちなみに

```javascript
$(function() {
	// 引数2つで $.when ----- A
	$.when(
		$("#nodeA").animate({left:300}),
		$("#nodeB").animate({left:300})
	).done(function() {
		console.log(arguments);
	});
	
	// 引数1つで $.when ----- B
	$.when(
		$("#nodeA").animate({left:300})
	).done(function() {
		console.log(arguments);
	});
});
```

```json:Aの出力
{
  "0": {
    "0": {},
    "length": 1,
    "context": {/*長いので略*/},
    "selector": "#nodeA"
  },
  "1": {
    "0": {},
    "length": 1,
    "context": {/*長いので略*/},
    "selector": "#nodeB"
  }
}
```

```json:Bの出力{
  "0": {
    "0": {},
    "length": 1,
    "context": {/*長いので略*/},
    "selector": "#nodeA"
  }
}
```

このように Ajax 系ではなくノードの操作系だときちんと想定する引数が渡ってきました。
なんだかよく分かりません。

