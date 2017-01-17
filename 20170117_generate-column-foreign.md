# generated column ＋ 外部キーをいろいろ試してみた

※ mysql 5.7.10 で試しています

## 準備

実験用テーブルを用意します。

```
DROP TABLE IF EXISTS table3;
DROP TABLE IF EXISTS table2;
DROP TABLE IF EXISTS table1;

CREATE TABLE table1 (
  id INT(11) NOT NULL AUTO_INCREMENT,
  fkval1 INT(11) NULL DEFAULT NULL,
  PRIMARY KEY (id),
  INDEX idx_table1 (fkval1)
);

CREATE TABLE table2 (
  id INT(11) NOT NULL AUTO_INCREMENT,
  val INT(11),
  fkval2 INT(11) AS (val * 2) STORED,
  PRIMARY KEY (id),
  INDEX idx_table2 (fkval2),
  CONSTRAINT fk1 FOREIGN KEY (fkval2) REFERENCES table1 (fkval1)
);

CREATE TABLE table3 (
  id INT(11) NOT NULL AUTO_INCREMENT,
  fkval3 INT(11) NULL DEFAULT NULL,
  PRIMARY KEY (id),
  CONSTRAINT fk2 FOREIGN KEY (fkval3) REFERENCES table2 (fkval2) ON UPDATE CASCADE ON DELETE CASCADE
);
```

`table1.fkval1` -> `table2.fkval2` -> `table3.fkval3` という親子関係です。
`table2.fkval2` は `val` を元にした generated column です(2倍の値)。
`id` はただの癖なので特に意味はありません。

## 実験(成功)

これらのテーブルにレコードを挿入してみます。

```
INSERT INTO table1 (fkval1) VALUES (10);
INSERT INTO table2 (val) VALUES (5);
INSERT INTO table3 (fkval3) VALUES (10);
```

特にエラーは出ません。

table1 に制約は存在しないため、fkval1 はなんでも入ります。ここでは 10 を入れています。

table2 は table1 への外部キー制約が存在するため、fkval2 の値は ↑で入れた 10 の値でなければなりません。
エラーはあとで試すのでとりあえず入るであろう 5 を入れています。

table3 は table2 への外部キー制約が存在するため、fkval3 の値は ↑で入れた 10 の値でなければなりません。
エラーはあとで試すのでとりあえず入るであろう 10 を入れています。

## 実験(失敗)

次に外部キー制約に引っかかるようなレコードを入れてみます。

```
INSERT INTO table2 (val) VALUES (10);
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`test`.`table2`, CONSTRAINT `fk1` FOREIGN KEY (`fkval2`) REFERENCES `table1` (`fkval1`))
```

怒られました。
table2.val に 10 を突っ込むと fkval2 が20になり、外部キーに違反するからです。
これにより、 generated column を外部キーの参照側にしても正常に動作していることがわかります。

```
INSERT INTO table3 (fkval3) VALUES (20);
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`test`.`table3`, CONSTRAINT `fk2` FOREIGN KEY (`fkval3`) REFERENCES `table2` (`fkval2`))
```

怒られました。
table2.fkval2 に 20 なんてレコードはないため、 table3.fkval3 に 20 を突っ込もうとして外部キー違反になるからです。
これにより、 generated column を外部キーの被参照側にしても正常に動作していることがわかります。

逆に言えば `INSERT INTO table1 (fkval1) VALUES (20)` を実行した後は上の2クエリは成功することになります。
実験(成功)と意味的には同じですが一応試しておきます。

```
INSERT INTO table1 (fkval1) VALUES (20);
Query OK, 1 row affected (0.00 sec)
INSERT INTO table2 (val) VALUES (10);
Query OK, 1 row affected (0.00 sec)
INSERT INTO table3 (fkval3) VALUES (20);
Query OK, 1 row affected (0.00 sec)
```

OK のようです。

## 参照アクション

ON UPDATE, ON DELETE を省略（RESTRICT）してるので他の参照アクションを試してみます。

```
ALTER TABLE table2 DROP FOREIGN KEY fk1;
ALTER TABLE table2 ADD CONSTRAINT fk1 FOREIGN KEY (fkval2) REFERENCES table1 (fkval1) ON UPDATE CASCADE ON DELETE CASCADE;
ERROR 3104 (HY000): Cannot define foreign key with ON UPDATE CASCADE clause on a generated column.
```

怒られました。CASCADE にはできないようです。

http://dev.mysql.com/doc/refman/5.7/en/innodb-foreign-key-constraints.html に記載がありますが、確かに試してみたところ、RESTRICT か NOT ACTION しか設定できないようです。
「ON UPDATE CASCADE はダメだよ！」とも言っているので、 ON DELETE だけを試したところ普通に行けるようでした。

```
ALTER TABLE table2 ADD CONSTRAINT fk1 FOREIGN KEY (`fkval2`) REFERENCES `table1` (`fkval1`) ON UPDATE RESTRICT ON DELETE CASCADE;
Query OK, 2 rows affected (0.10 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

generated column を外部キーの参照側にすることはめったに無いと思うし、個人的には ON UPDATE は割りとどうでもよくて ON DELETE の方が大事だと思うのでこれでも十分嬉しかったりします。
一応効いているか試してみます。

```
SELECT
  (SELECT COUNT(*) FROM table1) as table1count,
  (SELECT COUNT(*) FROM table2) as table2count,
  (SELECT COUNT(*) FROM table3) as table3count;
+-------------+-------------+-------------+
| table1count | table2count | table3count |
+-------------+-------------+-------------+
|           2 |           2 |           2 |
+-------------+-------------+-------------+

DELETE FROM table1;

SELECT
  (SELECT COUNT(*) FROM table1) as table1count,
  (SELECT COUNT(*) FROM table2) as table2count,
  (SELECT COUNT(*) FROM table3) as table3count;
+-------------+-------------+-------------+
| table1count | table2count | table3count |
+-------------+-------------+-------------+
|           0 |           0 |           0 |
+-------------+-------------+-------------+
```

最上位の table1 を吹き飛ばすと全て吹き飛んでいるのがわかります。

## 結論というか感想

「（個人的には） generated column はほぼ普通のカラムと同じように外部キー制約を活用できる」

都道府県コードと市区町村コードのように、「親子関係があり、かつ子キー値が親キー値を内包している」ときにいい感じに出来るかもしれません。
例えば東京都コードは「13」で、千代田区コード「13101」ですが、これのテーブル定義、いつも悩むんですよ。。。

- t_pref
  - pref_cd: 13
  - pref_name: 東京都
- t_city
  - city_cd: 13101
  - city_name: 千代田区

という構造だと外部キー制約が貼れないのでつらみがある。

- t_pref
  - pref_cd: 13
  - pref_name: 東京都
- t_city
  - pref_cd: 13
  - city_cd: 101
  - city_name: 千代田区

だと外部キー制約は貼れるけど、13101 という値を得るのに一手間必要だし、t_city を参照したいトランザクションテーブルで外部キーのために2カラム必要になる。

- t_pref
  - pref_cd: 13
  - pref_name: 東京都
- t_city
  - pref_cd: 13
  - city_cd: 13101
  - city_name: 千代田区

だと外部キーは貼れるし、13101 という値も直に得られるけど、13 を冗長に持ってることになる。

そこで2番目の方法で CONCAT(pref_cd, city_cd) (場合に応じて LPAD も必要)で generated column すればいい。
city_cd が必要なテーブルではその generated colum に対して外部キーを貼れば単一カラムで済む。

まぁ city_cd を持つようなテーブルは得てして pref_cd も持つので意味ないんだけどね…。まぁ例ということで。

