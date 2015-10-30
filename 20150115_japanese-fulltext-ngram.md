# InnoDB のフルテキストインデックスで日本語 NGRAM

「使い物になんねぇ」って印象でしたが、ちょっと試してみると簡単なキーワード検索程度なら十分実用的な気がします。

試行錯誤の結果を記します。

※ この文章は実験しつつ記述しています。整合性や内容の保証はできません
※ この文章はセキュリティ的なことを一切意識していません
※ 「5.6 からフルテキストインデックスが InnoDB でも使えるようになった」だけであり、基本的な仕様・動作は特に変わっていないはずです。設定項目名が変わっている（ft_min_word_len → innodb_ft_min_token_size 等）ようですがここでは触れません

## 事前作業

```sql
CREATE TABLE `article` (
	`seq` INT(11) NOT NULL AUTO_INCREMENT COMMENT '連番',
	`title` VARCHAR(64) NULL DEFAULT NULL COMMENT 'タイトル' COLLATE 'utf8_bin',
	`content` TEXT NULL COMMENT '本文' COLLATE 'utf8_bin',
	`bigram` TEXT NULL COMMENT 'bigram分かち書き' COLLATE 'utf8_bin',
	PRIMARY KEY (`seq`),
	FULLTEXT INDEX `idx_bigram` (`bigram`)
) CHARSET='utf8' ENGINE=InnoDB
```

上記のようなテーブルと

```sql
DELIMITER //
CREATE FUNCTION `NGRAM`(`tText` TEXT, `n` INT)
	RETURNS text
	DETERMINISTIC
BEGIN
	DECLARE tResult       TEXT;
	DECLARE nLength       INT;
	DECLARE nPosition     INT;
	DECLARE tPart         VARCHAR(16);

	IF tText IS NULL THEN
		RETURN NULL;
	END IF;

	SET tResult = '';

	SET tText = TRIM(REPLACE(tText, '　', ''));
	SET nLength = CHAR_LENGTH(tText);

	SET nPosition = 1;
	WHILE nPosition <= nLength DO
		SET tPart = TRIM(SUBSTR(tText, nPosition, n));
		IF CHAR_LENGTH(tPart) > 0 THEN
			SET tResult = CONCAT(tResult, ' ', tPart);
		END IF;
		SET nPosition = nPosition + 1;
	END WHILE;

	RETURN TRIM(tResult);
END
//
```

上記のようなファンクションと

```sql
DELIMITER //
CREATE TRIGGER `trg_article_insert` BEFORE INSERT ON `article` FOR EACH ROW BEGIN
    SET NEW.bigram = CONCAT(NGRAM(NEW.title, 2), '　', NGRAM(NEW.content, 2));
END
CREATE TRIGGER `trg_article_update` BEFORE UPDATE ON `article` FOR EACH ROW BEGIN
    SET NEW.bigram = CONCAT(NGRAM(NEW.title, 2), '　', NGRAM(NEW.content, 2));
END
//
```

上記のようなトリガーと

```txt:MySQL
MySQL（マイエスキューエル）は、オラクルが開発するRDBMS（リレーショナルデータベースを管理、運用するためのシステム）の実装の一つである。

オープンソースで開発されており、GNU GPLと商用ライセンスのデュアルライセンスとなっている。

MySQLは GPL とコマーシャルライセンスのデュアルライセンス方式で提供されている。 基本的に、MySQLのサーバ本体とクライアントライブラリはGPLで提供される。このため、MySQLを改造し、それを再頒布する場合は、GPLに従う必要がある。
```

```txt:PostgreSQL
PostgreSQL（ぽすとぐれすきゅーえる: 発音例）は、BSDライセンスに類似するライセンスにより配布されているオープンソースのオブジェクト関係データベース管理システム (ORDBMS) である。その名称は Ingres の後継を意味する「Post-Ingres」に由来している。単純に「Postgres」や「ポスグレ」と呼称されることも多い。

PostgreSQL はバージョン 9.0 よりレプリケーションを標準でサポートするが、サードパーティー製のオプション・ソフトウェアも利用できる。
Fermion、検索および更新処理の負荷分散、自動フェイルオーバー機能、マルチキャストを用いたノードの自動追加処理機能を備える。
```

```txt:Microsoft_SQL_Server
Microsoft SQL Server （マイクロソフト エスキューエル サーバ）とは、マイクロソフトが開発している、リレーショナルデータベース管理システム (RDBMS)である。略称は「SQL Server」または「MS SQL」などと呼ばれている。主要な問い合わせ言語 (クエリ言語)は、T-SQLとANSI SQLである。

企業サーバ向けの高機能なシステムから、組み込み系の小規模なシステムまで幅広く対応する。またMicrosoft Windowsと親和性が高く、ADOやADO.NETを経由して最適なバックエンドデータベースを構築できるようになっている。
```

上記のようなデータを用意します。
実際には他に多少レコードを入れて実験します。

簡単に説明すると、

テーブルは `{連番, タイトル, 本文, タイトル・本文を BIGRAM 化して結合したもの}` で構成されます。bigram カラムには fulltext インデックスが張られています。

NGRAM ファンクションは引数の文字列をスペース区切りの NGRAM で返します（参考 [http://d.hatena.ne.jp/Sikushima/20130610/1370866252](http://d.hatena.ne.jp/Sikushima/20130610/1370866252) ）。

トリガーはレコード追加・変更時に bigram カラムへデータを突っ込むためです。

データは適当に wikipedia から拾ってきました。内容は変更してませんが、良い例示になるようにセンテンスを抜粋してあります。

* [MySQL](http://ja.wikipedia.org/wiki/MySQL)
* [PostgreSQL](http://ja.wikipedia.org/wiki/PostgreSQL)
* [Microsoft SQL Server](http://ja.wikipedia.org/wiki/Microsoft_SQL_Server)


上記の事前作業により、データ作成時・更新時にトリガーで bigram カラムに

```
オラ ラク クル ルが が開 開発 発す する るR RD DB BM MS S（抜粋
その の名 名称 称は は I In ng gr re es s の の後 後継 継を を意 意味 味す する（抜粋
企業 業サ サー ーバ バ向 向け けの の高 高機 機能 能な なシ シス ステ テム ムか から（抜粋
```

のようなデータが作成されています。

## 検索

要件・仕様としては「`article` テーブルの `title` or `content` から全文検索したい」とします。

まず「サーバ」という単語で普通に LIKE 検索を行ってみます。

```
mysql> SELECT seq FROM article WHERE (title LIKE '%サーバ%') OR (content LIKE '%サーバ%');
+-----+
| seq |
+-----+
|   1 |
|   3 |
+-----+

mysql> EXPLAIN SELECT seq FROM article WHERE (title LIKE '%サーバ%') OR (content LIKE '%サーバ%');
+----+-------------+---------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table   | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+---------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | article | ALL  | NULL          | NULL | NULL    | NULL |   57 | Using where |
+----+-------------+---------+------+---------------+------+---------+------+------+-------------+
```

2 件でした。1 と 3 には「サーバ」という文字列が含まれているので当たり前です。
ただし、当然のごとく実行計画が劣悪です。

次に MATCH AGAINST 検索を行ってみます。

```
mysql> SELECT seq FROM article WHERE (MATCH (bigram) AGAINST (NGRAM('サーバ', 2)));
+-----+
| seq |
+-----+
|   3 |
|   1 |
|   2 |
|   4 |
|   9 |
|   6 |
|   7 |
+-----+

mysql> EXPLAIN SELECT seq FROM article WHERE (MATCH (bigram) AGAINST (NGRAM('サーバ', 2)));
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
| id | select_type | table   | type     | possible_keys | key        | key_len | ref  | rows | Extra       |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
|  1 | SIMPLE      | article | fulltext | idx_bigram    | idx_bigram | 0       | NULL |    1 | Using where |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
```

何やらいっぱいヒットしました。
説明は省きますが、きっちりとした検索をしたい場合は BOOLEAN MODE と呼ばれるものでないとなかなか使いにくいです。

というわけで BOOLEAN MODE で試してみます。

```
mysql> SELECT seq FROM article WHERE (MATCH (bigram) AGAINST ('+サー +ーバ' IN BOOLEAN MODE));
+-----+
| seq |
+-----+
|   3 |
|   1 |
|   2 |
+-----+

mysql> EXPLAIN SELECT seq FROM article WHERE (MATCH (bigram) AGAINST ('+サー +ーバ' IN BOOLEAN MODE));
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
| id | select_type | table   | type     | possible_keys | key        | key_len | ref  | rows | Extra       |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
|  1 | SIMPLE      | article | fulltext | idx_bigram    | idx_bigram | 0       | NULL |    1 | Using where |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
```

3 件引っかかりました。LIKE 検索と結果が異なります(ちなみに「+」記号は「必ず含む」を意味します)。
これは「サー」「ーバ」を両方含むレコードがヒットしているからです。
データを見てみると「**サー**ドパーティー製」「フェイルオ**ーバ**ー機能」という文字列があり、これらのせいでヒットしていると考えられます。
欲しいのは「サーバ」を含むものなのでこれらは除外したいです。

そこで MATCH AGAINST と LIKE の合わせ技を使います。

```
mysql> SELECT seq FROM article WHERE (MATCH (bigram) AGAINST ('+サー +ーバ' IN BOOLEAN MODE) AND ((title LIKE '%サーバ%') OR (content LIKE '%サーバ%')));
+-----+
| seq |
+-----+
|   3 |
|   1 |
+-----+

mysql> EXPLAIN SELECT seq FROM article WHERE (MATCH (bigram) AGAINST ('+サー +ーバ' IN BOOLEAN MODE) AND ((title LIKE '%サーバ%') OR (content LIKE '%サーバ%')));
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
| id | select_type | table   | type     | possible_keys | key        | key_len | ref  | rows | Extra       |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
|  1 | SIMPLE      | article | fulltext | idx_bigram    | idx_bigram | 0       | NULL |    1 | Using where |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
```

オプティマイザが変な判断を下さなければ、フルテキストインデックスが適切に使用された後の結果セットを LIKE で絞り込むような動作になるはずです。
これで「フルテキストインデックスを使用しつつ、余計な行を含まない検索」が実現できました。

が、これではまだ不完全です。
せっかく全文検索を用いたいのに LIKE なんて書きたくありません。
それに仕様変更などで対象カラムが増えると LIKE が増えて見通しが悪くなりますし、修正漏れもあるかもしれません。要するに「このカラムで全文検索」という前提は崩したくないです（まぁ仕様変更の場合、トリガーの再生成・再実行が必要ですがこの際考えないことにします）。
また、全文検索の結果が大きいとそれら全てに LIKE 検索しそうなので、パフォーマンス上の懸念もあります。

そこで BOOLEAN MODE の検索オプションに「フレーズ検索」というものがあるのでそれを使ってみます。

```
mysql> SELECT seq FROM article WHERE (MATCH (bigram) AGAINST ('+"サー ーバ"' IN BOOLEAN MODE));
+-----+
| seq |
+-----+
|   3 |
|   1 |
+-----+

mysql> EXPLAIN SELECT seq FROM article WHERE (MATCH (bigram) AGAINST ('+"サー ーバ"' IN BOOLEAN MODE));
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
| id | select_type | table   | type     | possible_keys | key        | key_len | ref  | rows | Extra       |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
|  1 | SIMPLE      | article | fulltext | idx_bigram    | idx_bigram | 0       | NULL |    1 | Using where |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
```

LIKE なしで結果が LIKE 検索と一致しました。
`+"サー ーバ"` という表記がミソのようです。ちょっと情報が足りないんでどういう動作かはわからないんですが、どうも連続した順番を意識した検索を行ってくれるような印象です。

なお、各 bigram の単語に「+」修飾子を付ける検索とは違い、全体に「+」を付ければいいだけなので、定義した `NGRAM` ファンクションが使用できます。

```
mysql> SELECT seq FROM article WHERE (MATCH (bigram) AGAINST (CONCAT('+', '"', NGRAM('サーバ', 2), '"') IN BOOLEAN MODE));
+-----+
| seq |
+-----+
|   3 |
|   1 |
+-----+

mysql> EXPLAIN SELECT seq FROM article WHERE (MATCH (bigram) AGAINST (CONCAT('+', '"', NGRAM('サーバ', 2), '"') IN BOOLEAN MODE));
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
| id | select_type | table   | type     | possible_keys | key        | key_len | ref  | rows | Extra       |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
|  1 | SIMPLE      | article | fulltext | idx_bigram    | idx_bigram | 0       | NULL |    1 | Using where |
+----+-------------+---------+----------+---------------+------------+---------+------+------+-------------+
```

ダブルクオートしか受け入れてくれないのでちょっと不格好なことをしていますが、少なくとも "サー"、"ーバ"みたいな変な文字は SQL 文中から消すことができました。

あと、これだと `title` と `content` の境目でヒットする気がしないでもない（上記で言えば `title` が「サー」で終わり、`content` が「ーバ」で始まるレコード ）ですが、トリガーを工夫すれば避けられると思います。
上記トリガーではカラムの区切り文字に全角スペースを使用してヒットしないようにしています。

ここまで来るとアプリレイヤーで検索語を ngram 化したりする必要がなくなり、全文検索をほとんど意識せずに済みます。

## まとめ

まだまだ検証不足ですが、それなりに実用的な機能なんじゃないでしょうか。
InnoDB なのでトランザクションが使えるのは非常に大きいです。

機会があれば積極的に使ってみたいと思います。
ただデータサイズが 3 倍程度になったり、トリガーやファンクションなど、本筋とは異なることをしなければならないのがネックかもしれません。
