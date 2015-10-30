# GROUP BY を使用せずに HAVING を使う

例えば下記のように複雑な条件に基づいてレコードを抽出する場合、WHERE を使うと残念なことになります。

```sql
SELECT
    T.id,
    (
        CASE
            WHEN /*難解極まりない条件1*/ false THEN 1
            WHEN /*難解極まりない条件2*/ false THEN 2
            WHEN /*難解極まりない条件3*/ false THEN 3
            ELSE 0
        END
    ) AS stat
FROM
    tbl T
WHERE
    (
        CASE
            WHEN /*難解極まりない条件1*/ false THEN 1
            WHEN /*難解極まりない条件2*/ false THEN 2
            WHEN /*難解極まりない条件3*/ false THEN 3
            ELSE 0
        END
    ) = 2
```

とても保守性が低いと思います。
WHERE を無くして `stat` を使用してアプリケーションレイヤーで絞り込むことも出来ますが、そうするとページング処理等がやりたい場合、とても非効率なことになります(データベースレイヤーでの LIMIT が使用できないため)。

こんなとき、HAVING 句を使うとエイリアス名が使えるため、シンプルに記述することが出来ます。

```sql
SELECT
    T.id,
    (
        CASE
            WHEN /*難解極まりない条件1*/ false THEN 1
            WHEN /*難解極まりない条件2*/ false THEN 2
            WHEN /*難解極まりない条件3*/ false THEN 3
            ELSE 0
        END
    ) AS stat
FROM
    tbl T
HAVING
    stat = 2
```

どうも GROUP BY がなくても HAVING 句が使える模様。

※ もともと件数が少ない or インデックスが使われないであろう状況を想定
※ mysql 5.5.36 で実験。他のバージョン、RDBMS は分からない
