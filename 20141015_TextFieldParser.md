# Microsoft.VisualBasic.FileIO.TextFieldParser は空行を読み飛ばす

csv の詳しい仕様は知りませんが、とりあえずそのような挙動を取るようです。

[MSDN：TextFieldParser.ReadFields メソッド ](http://msdn.microsoft.com/ja-jp/library/microsoft.visualbasic.fileio.textfieldparser.readfields%28v=vs.110%29.aspx)

> ReadFields が空白行を検出した場合は、空白行がスキップされて次の空白でない行が返されます。

-----

例えば下記のような CSV ファイル

```test.csv
"hoge


fuga"
```

ちょっと分かりにくいですが、1行1列の CSV ファイルです。唯一のフィールドの文字列 hoge⇔fuga 間に空行があります。

このファイルを TextFieldParser を使用して出力すると…

```csharp
var textParser = new Microsoft.VisualBasic.FileIO.TextFieldParser("/path/to/test.csv");
textParser.SetDelimiters(",");

while (!textParser.EndOfData)
{
	var fields = textParser.ReadFields();
	Console.WriteLine(fields[0]);
}
```

下記が出力されました。

```
hoge
fuga
```

設定するメソッドなどはないらしい。
なんだかとってももったいない気持ち。
