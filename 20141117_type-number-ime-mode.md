# input[type=number] は ime-mode が効かない

Windows, Firefox(31.1)のみで確認。

```html
<input name="test" type="number" style="ime-mode:disabled;">
```

という html でも Firefox だと IME をポチポチ切り替えることが出来て、全角数字を入力することが出来ます。
（number を text に変えると切り替えられない）

全角数字入力状態で横っちょのスピンボタンをクリックすると半角に変換されます。
全角数字以外が入力されているとスピンボタンが効かないようです。

そして入力の全半角を問わず、サーバーには半角で飛んできます。

「ime-mode が効かねぇeeeee」って人の参考になれば幸い。
まぁ大した問題にはならないと思いますが、個人的には数値は半角で入力しないと気持ち悪い…。

わざわざ ime-mode を無視する理由がなにかあったんかね？
むしろ [type=number] だったら強制 IME オフでもいいような…。
