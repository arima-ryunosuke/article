# まだ日本語全文検索で消耗してるの？

この記事は [InnoDB のフルテキストインデックスで日本語 NGRAM](http://qiita.com/ArimaRyunosuke/items/d2b3b94f223cb83c463d) の続きです。
以降↑の記事を「前回の記事」と呼称します。

例によって実験しつつ記述しています。整合性や内容の保証はできません。
検証に使ったのは CentOS 7, mysql 5.7.9 です。


## 前回の記事は何をしているのか

端的に言えば下記です。

1. 文字列を ngram 化するファンクションを定義
2. 全文検索したい複数カラムを結合して ngram 化した文字列を格納するカラムを定義
3. トリガーで↑のカラムに ngram 化した文字列を放り込む
4. ↑↑のカラムに対して FULLTEXT INDEX を張る
5. 検索時に ↑↑↑のカラムに対して MATCH AGAINST 検索を行うことで全文検索

とまぁ色々めんどいことをしています。
特に本筋ではないトリガーとファンクションの定義が嫌。


## mysql 5.7.9 には・・・

ところで mysql 5.7.9 には下記の機能があります。

1. 組み込みの ngram パーサー
2. generated column

この2つの機能だけで上記の 1, 2, 3 の機能を兼ねています。
具体的には

- 組み込みの ngram パーサーがあるのでオレオレ ngram ファンクションの定義は不要
- generated column により、自動的に複数カラムを結合したカラムが定義可能
- ↑で定義した自動カラムに対してインデックスを貼ることが出来る

ことに起因します。

なお、組み込み ngram の `N` は `ngram_token_size` 変数で設定可能です。デフォルトは 2 のようです。


## 実践

と、いうわけで、前回の記事を mysql 5.7 に対応してみました。
前回と同じく、要件・仕様としては「article テーブルの title or content から全文検索したい」とします。


### 事前作業

まずテーブル作成。

```sql
CREATE TABLE article (
    seq INT(11) NOT NULL AUTO_INCREMENT COMMENT '連番',
    title VARCHAR(64) NULL DEFAULT NULL COMMENT 'タイトル',
    content TEXT NULL COMMENT '本文',
    fulltext_column TEXT AS (CONCAT(title, '　', content)) STORED,
    PRIMARY KEY (seq),
    FULLTEXT INDEX ftx_fulltext (fulltext_column) /*!50100 WITH PARSER `ngram` */ 
)COLLATE='utf8_bin' ENGINE=InnoDB;
```

`fulltext_column` が全文検索用カラムです。
generated column なのでトリガーで突っ込んだり更新する必要はありません。
`STORED` なのは `VIRTUAL` だとなんか怖かったからです。
試してないですが、動くようなら `VIRTUAL` でも良いと思います。

`ftx_fulltext` が FULLTEXT INDEX です。↑の `fulltext_column` に対して貼ります。
``/*!50100 WITH PARSER `ngram` */`` がキモで、これによりトークナイズが ngram になります。

次にレコード挿入。

```sql
INSERT INTO article (title, content) VALUES('MySQL', 'MySQL（マイエスキューエル）は、オラクルが開発するRDBMS（リレーショナルデータベースを管理、運用するためのシステム）の実装の一つである。
オープンソースで開発されており、GNU GPLと商用ライセンスのデュアルライセンスとなっている。
MySQLは GPL とコマーシャルライセンスのデュアルライセンス方式で提供されている。 基本的に、MySQLのサーバ本体とクライアントライブラリはGPLで提供される。このため、MySQLを改造し、それを再頒布する場合は、GPLに従う必要がある。');

INSERT INTO article (title, content) VALUES('PostgreSQL', 'PostgreSQL（ぽすとぐれすきゅーえる: 発音例）は、BSDライセンスに類似するライセンスにより配布されているオープンソースのオブジェクト関係データベース管理システム (ORDBMS) である。その名称は Ingres の後継を意味する「Post-Ingres」に由来している。単純に「Postgres」や「ポスグレ」と呼称されることも多い。
PostgreSQL はバージョン 9.0 よりレプリケーションを標準でサポートするが、サードパーティー製のオプション・ソフトウェアも利用できる。
Fermion、検索および更新処理の負荷分散、自動フェイルオーバー機能、マルチキャストを用いたノードの自動追加処理機能を備える。');

INSERT INTO article (title, content) VALUES('Microsoft SQL Server', 'Microsoft SQL Server （マイクロソフト エスキューエル サーバ）とは、マイクロソフトが開発している、リレーショナルデータベース管理システム (RDBMS)である。略称は「SQL Server」または「MS SQL」などと呼ばれている。主要な問い合わせ言語 (クエリ言語)は、T-SQLとANSI SQLである。
企業サーバ向けの高機能なシステムから、組み込み系の小規模なシステムまで幅広く対応する。またMicrosoft Windowsと親和性が高く、ADOやADO.NETを経由して最適なバックエンドデータベースを構築できるようになっている。');
```

前回の記事の「テーブル作成」と「データの用意」の部分だけです（データ部分は wikipedia から引用 [MySQL](http://ja.wikipedia.org/wiki/MySQL),[PostgreSQL](http://ja.wikipedia.org/wiki/PostgreSQL),[Microsoft SQL Server](http://ja.wikipedia.org/wiki/Microsoft_SQL_Server) ）。
トリガの作成とかファンクションの定義とかは行いません、不要です。


### 検索

前振りはおいておいて、いきなり検索を試してみます。

```
mysql> SELECT seq FROM article WHERE (MATCH (fulltext_column) AGAINST ('"サーバ"' IN BOOLEAN MODE)); 
+-----+
| seq |
+-----+
|   3 |
|   1 |
+-----+
2 rows in set (0.00 sec)

mysql> EXPLAIN SELECT seq FROM article WHERE (MATCH (fulltext_column) AGAINST ('"サーバ"' IN BOOLEAN MODE))\G                                                                                                          
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: article
   partitions: NULL
         type: fulltext
possible_keys: ftx_fulltext
          key: ftx_fulltext
      key_len: 0
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where; Ft_hints: no_ranking
1 row in set, 1 warning (0.00 sec)
```

日本語で全文検索できました。楽すぎる。

※ こうしてみると前回の SELECT 文間違ってますね…


## 懸念とか

### 意図せず大量にデータ領域を使いそう

`STORED` なので title と content を結合した（自動的な）カラムが実際のデータとして格納され、さらにそれに対して ngram 化した文字列（約3倍？）がインデックスとして格納されます。
なので実質的には4倍程度のデータ量になりそうです。
まぁ自分で定義しているので「意図せず」にはなりませんが…。

なお fulltext_column を `VIRTUAL` にすればインデックス領域だけで済みそうです。
（ちなみに `VIRTUAL` に対してもインデックスは貼れます。実データとしては持たずにインデックスとしてだけ持つようです）。


### インデックスの再構築が遅いらしい

ソースは http://yoku0825.blogspot.jp/2015/03/mysql-576innodb-ngram.html です。

`{タイトル, 本文}` からの全文検索をしてますが、これが仕様変更によりカラムが増えるとインデックスの再構築が走ります。
したがって、上記の記事の「とても遅い再構築」が走ることになるわけです。
カラムがめったに変わらないなら使い物になる…かな？


### 「ランク付けして」とか言われたら死ねる

ngram と違って mecab 組み込みパーサがないから。
まぁ入れればいいんですが…。できればバニラな状態でいろいろやりたくなるのが人情でしょう。


## まとめ

ちょっと懸念はあるけど、あまりある有用性と手軽さがある（と、個人的には思います）。
