# 行値式の IN でインデックスが使われない

mysql 5.5.36、下記のような行値式を使ったクエリでインデックスが使われませんでした。
便利な構文ですが、使用するときは注意した方がいいかもしれません。

## 前提

```sql
CREATE TABLE `tbl` (
    `id` INT(11) NOT NULL,
    `seq` INT(11) NOT NULL,
    `data` TEXT NOT NULL,
    PRIMARY KEY (`id`, `seq`)
) ENGINE=InnoDB;
```

このようなテーブルで

```sql
+----+-----+-------+
| id | seq | data  |
+----+-----+-------+
|  1 | 100 | dummy |
|  2 | 200 | dummy |
|  3 | 300 | dummy |
+----+-----+-------+
```

このようなレコードを取得したいとします。
なお、レコードは 10000 行くらい突っ込んであります。

## 普通に OR で繋げた場合

```sql
explain select * from tbl where (id = 1 and seq = 100) or (id = 2 and seq = 200) or (id = 3 and seq = 300);
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | tbl   | range | PRIMARY       | PRIMARY | 8       | NULL |    3 | Using where |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
```

主キーが使用されています。

## 行値式の場合

```sql
explain select * from tbl where (id, seq) in ((1, 100), (2, 200), (3, 300));
+----+-------------+-------+------+---------------+------+---------+------+-------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows  | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+-------+-------------+
|  1 | SIMPLE      | tbl   | ALL  | NULL          | NULL | NULL    | NULL | 10297 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+-------+-------------+
```

全行読み込んでるっぽいです。
ちなみに IN を使わなければ使用されるようです。

```sql
explain select * from tbl where (id, seq) = (1, 100) or (id, seq) = (2, 200) or (id, seq) = (3, 300);
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | tbl   | range | PRIMARY       | PRIMARY | 8       | NULL |    3 | Using where |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
```

でもあんまり使ってるメリットを感じない…。