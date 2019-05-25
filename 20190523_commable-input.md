# value を変えずに input に3桁カンマを振る

「数値に3桁カンマを振りたい」って要望は結構あって、ググると js で value 自体をゴリゴリいじる実装が多いですが、今回はちょっと違ったアプローチをしてみます。

## コード

```html
<html lang="ja">
<head>
    <meta http-equiv="content-type" content="text/html; charset=utf-8"/>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/twitter-bootstrap/3.3.7/css/bootstrap.min.css">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
</head>
<body>

<style>
    /* number-input と number-overlap をまとめるコンテナ */
    .number-container {
        position: relative;
    }

    /* カンマ区切りにしたい input 要素 */
    .number-input {
        position: relative;
    }
    /* 普段は z-index:1 で number-overlap の後ろに居る。色は被ってしまうので transparent */
    .number-input:not(:focus) {
        z-index: 1;
        color: transparent;
    }
    /* フォーカス中は z-index:3 で number-overlap の前に来る。色も戻す */
    .number-input:focus {
        z-index: 3;
        color: #555;
    }

    /* number-input に覆い被さる要素。 pointer-events:none でクリックを透過させる */
    .number-overlap {
        position: absolute;
        top: 1px;
        left: 0px;
        display: inline-block;
        background: transparent;
        width: 100% !important;

        pointer-events: none;
        z-index: 2;
    }
</style>
<script>
    $(function () {
        // フォーカスを抜けたり値が変わったりしたら覆い被さっているテキストをカンマ区切りにする
        $('#hoge').on('blur change input', function () {
            const $this = $(this);
            if ($this.val() !== '') {
                $this.next().text((+$this.val()).toLocaleString());
            }
        });
    });
</script>

<form class="form-inline">
    <div class="form-group">
        <div class="number-container">
            <input type="text" id="hoge" name="hoge" class="form-control number-input" placeholder="自動でカンマが付きます">
            <span class="number-overlap form-control"></span>
        </div>
    </div>
</form>

</body>
</html>
```

殴り書きしたのでかなり荒いですが、コメントを読めば何をしているかは分かると思います。
一応説明を加えると下記のような感じです。

### .number-container

`position` で要素を重ね合わせる必要があるので、そのためのコンテナです。
それ以上の意味は特にありません。

### .number-input

`input` に付与します。
普段は `z-index` で後述の `.number-overlap` の後ろに隠れていますが、フォーカスすると前面に出てきます。

### .number-overlap

実際に3桁カンマの文字列が設定される要素です。
普段は `z-index` で前述の `.number-input` に被さっていますが、フォーカスされると後ろに引っ込みます。
`pointer-events:none` なので**クリックを透過します**。これが今回のキモです。これによって「被さっているんだけど後ろの要素がクリック（フォーカス）できる」が実現できます。

### javascript

フォーカスとか見え方とかは css だけで完結しますが、「カンマを振る処理そのもの」はどうしても js の力を借りる必要があるのでそれをしています。
できれば html + css だけでやりたい・・・。

## 注意点

ただし上記はかなりサボっています。実際はいろいろ調整しないと使い物になりません。

まず `.number-overlap` のスタイルは `number-input` に完全に合わせる必要があります（padding や font-size など多岐にわたる。合わせないとすぐズレる）。
上記は bootstrap を使っていますが `form-control` は `input` に限定されないので流用してるだけです。しっかりやろうとすると結構めんどくさいです。
実際、上記は chrome で確認しましたが、他のブラウザではズレたりします。そういった微調整が必須になります。

また、個人的に数値入力は `type=number` を使うことが多いです。さらに右寄せにしたいですが、スピンボタンがくっつくのでブラウザごとの調整が必要です（幅がブラウザごとに全く異なります）。
正直かなり大変なので、いっその事スピンボタンは消したほうが良いです。

js サイドも上のコードだと数値以外を入れると NaN になるので、もっといろいろな分岐が必要です。
NaN は `type=number` でサボったり出来ますが `e` の罠があったりして、正直「ロケール指定ができる `type=number` があればいいのに・・・」と思います。

## まとめ

注意点で記載した所が調整・クリアできれば変換の面倒をまったく見なくて済むようになるので結構有用だと思います。
（js で値をゴリゴリいじるのはなんか好かん）。
