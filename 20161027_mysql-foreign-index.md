# mysql5.6 では暗黙のインデックスが別管理になっている？

mysql では外部キー制約を作成すると暗黙のうちにインデックスが作成されますが、どうもバージョンによってそのインデックスの挙動が異なるようです。

```sql:mysql5.6
# 何度も実行したいので掃除
DROP TABLE IF EXISTS t_child;
DROP TABLE IF EXISTS t_parent;

# 親子テーブルを作成
CREATE TABLE t_parent (
    parent_id INT(11) NOT NULL AUTO_INCREMENT,
    PRIMARY KEY (parent_id)
);
CREATE TABLE t_child (
  child_id INT(11) NOT NULL AUTO_INCREMENT,
  parent_id INT(11) NOT NULL,
  PRIMARY KEY (child_id)
);

# インデックスを確認
SHOW INDEX FROM t_child;

# 外部キーを作成
ALTER TABLE t_child ADD CONSTRAINT fk_PrentChild FOREIGN KEY (parent_id) REFERENCES t_parent (parent_id);

# 外部キーを作成したので fk_PrentChild というインデックスが作られている
SHOW INDEX FROM t_child;

# ↑で見たインデックスと全く同じインデックスを作ってみる
CREATE INDEX fk_PrentChild ON t_child (parent_id);
/* Affected rows: 0  Found rows: 0  注意: 0  Duration for 1 query: 0.015 sec. */
# 何故か通った

# インデックスを確認（増えてない）
SHOW INDEX FROM t_child;

# もう一度作ってみる
CREATE INDEX fk_PrentChild ON t_child (parent_id);
/* SQL エラー  (1061): Duplicate key name 'fk_PrentChild' */
# 今度は怒られた
```

mysql5.5 だと1回目の `CREATE INDEX fk_PrentChild ON t_child (parent_id)` は実行できず、 `Incorrect index name` と言われます。
mysql5.6 だと上記の通り、全く同じインデックスを明示的に作っても1回目は通ります。そして2回目でエラーになります。
また、 5.6 でも`CREATE INDEX fk_hogera ON t_child (parent_id)` のようにすると最終的なインデックス数は2個（PRIMARY, fk_hogera）となり、fk_PrentChild が消えています。

これらのことと、下記の公式マニュアルの記述：

https://dev.mysql.com/doc/refman/5.6/ja/innodb-foreign-key-constraints.html
> このインデックスは、外部キー制約を適用するために使用できる別のインデックスを作成した場合、あとで暗黙のうちに削除される可能性があります。

から、確信は持てないですが「外部キーによる暗黙のインデックスは別管理されている」と結論づけました。
間違っていたら誰かご教示ください。
