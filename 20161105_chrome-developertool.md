# Chrome デベロッパーツールを開いているとPOST再送ダイアログが表示されない

題名の通りで、特筆すべきことは何もないんですが…。

```html:hoge.html
<form method="post">
<input type="text" name="hoge">
<input type="submit" value="ok">
</form>
```

このようなシンプルな html を適当な web サーバに置いて submit 後に F5してみると再現できます。
（何の警告もなくPOST再送される）

http://stackoverflow.com/questions/25527260/chrome-dev-tools-ignores-form-re-submit-warning

で「そっちのほうが便利じゃね？」って感じで推測されてるけど結構有名なのかな？
個人的には全然知らんくて「なんでだー」って感じで10分程度ハマったので備忘録として残しておきます。

