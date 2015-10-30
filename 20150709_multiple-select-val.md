# multiple な select の $.fn.val() は配列を返す

公式ドキュメントにも書かれていたんですが、少しハマったのでメモします。

タイトル通り、select-multiple の `val()` は配列を返します。

```html
<select id="select-multiple" multiple>
	<option value="1">オプション1</option>
	<option value="2">オプション2</option>
	<option value="3">オプション3</option>
</select>

<script type="text/javascript">
	$('#select-multiple').change(function() {
		console.log($(this).val());
	});
</script>
```

この html で select 要素のオプションを選択すると `[ "1" ]` のような配列が得られます。
Ctrl + クリック等で複数を選択すると `[ "1", "3" ]` です。便利ですね。

が、言いたいことはそんなことではなくて、「何も選択しなかった」時の戻り値です。

何も選択しなかった時の戻り値として `[]` （空配列）を期待したんですが、そんなことはなく、返ってきたのは **null** でした。version 1.11.1 です。
これって仕様なんですかね？ なんか一貫性がないような…。
