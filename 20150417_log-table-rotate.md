# mysql.general_log をローテート？する

## 何がしたいの？

[一般クエリとスロー クエリのログ出力先の選択](http://mysql.stu.edu.tw/doc/refman/5.1/ja/log-tables.html) に記載の通り、`log-output` を `TABLE` にすれば、クエリログが `mysql.general_log` テーブルに保存されるようになります。
ログを SQL で検索できるようになるのでそれはそれは死ぬほど便利なんですが、ファイルではないので logrotate が行われません。
スロークエリはともかく、一般クエリログは膨大なサイズに成り得ますので、できれば自動で削減したいです。

が、簡単に調べたところ、`mysql.general_log` と `mysql.slow_log` は「log tables」と呼ばれるちょっと特別扱いなテーブルらしく、能動的な INSERT/UPDATE/DELETE 処理が一切行えず、TRUNCATE のみが可能なようです。
定期的に TRUNCATE してもいいんですが、それだと「ログを SQL で検索できる」という利点がほとんどなくなってしまいます。

上記のリンクを読むと「TRUNCATE TABLE を使用して、ログ エントリに有効期限をつけることができる」というそれっぽい文言がありましたが、何を意味するのかよく分かりませんでした。

## 準備

クエリログをテーブルとファイルに出力するように my.cnf を設定します。

```ini:/etc/my.cnf
general_log      = 1
general_log_file = /var/log/mysql/query.log
log_output       = FILE,TABLE
```

mysqld を再起動し、適当なクエリを投げて `mysql.general_log` が増えていれば OK です。
なぜファイル/テーブルの両方に出力するのかは後述します。


次にログローテートの postrotate にパージ処理を挟みます。
ログローテートのタイミングで行う必然性は特にないんですが、データを常に「ファイルログ <= テーブルログ」にしたい（基本的にはテーブルを見れば OK にしたい）ので、記述が近いここで行います。

```plain:/etc/logrotate.d/mysqld
/var/log/mysql/query.log {
    create 640 mysql mysql
    daily
    rotate 30
    missingok
    sharedscripts
    postrotate
    # just if mysqld is really running
    if test -x /usr/bin/mysqladmin && \
       /usr/bin/mysqladmin ping &>/dev/null
    then
       /usr/bin/mysqladmin flush-logs
       
       COUNT=$(/usr/bin/mysql -u root -NB -e "SELECT COUNT(*) FROM mysql.general_log WHERE event_time <= DATE_ADD(NOW(), INTERVAL -30 DAY);")
       /bin/sed -e "1,${COUNT}d" -i /var/lib/mysql/mysql/general_log.CSV
       /usr/bin/mysql -u root -e "FLUSH TABLE mysql.general_log"
    fi
    endscript
}
```

何をしているかというと、デフォルトでは mysql.general_log は CSV ストレージエンジンなので、csv ファイルをいじれば TABLE データも変更されます。
さらに、所詮ログなので、時系列順に並んでいる（と予想される）ため、削除したい行数さえ分かれば sed でごそっと削除できます。

WHERE の条件はローテートに合わせます。ここでは毎日で30世代なので、現在より30日以前のログをカウントしています。
膨れ上がらない程度 かつ ファイルログより長めに保持されるのであればある程度適当でいいと思います。

ただこれはファイルの置換操作になる（と思われる）ので テーブルが掴んでいるファイルディスクリプタを切り替えるため、`FLUSH TABLE` が必要です。
さらにパフォーマンスもあまりよろしくないと思います。

## 確認

30 日だと確認しづらいので 1 分程度にして確認します。
上記の WHERE 部分を「INTERVAL -1 MINUTE」に変更し、適当にクエリログを出させた後、

```bash
logrotate -f /etc/logrotate.d/mysqld
mysql -u root -e "select * from mysql.general_log\G"
```

で１分前以前のクエリログが吹き飛んでいれば OK です。

上記はすべてクエリログでしたが、スロークエリでも基本的には同じです。

## これで大丈夫？

上記でクエリログがテーブルに保存され、かつ膨大にならない程度に自動的にパージされるようになります。
調査時などに SQL ベースでログが見れるのはとても大きいことだと思います。

ただし、sed から FLUSH TABLE の間のログは記録されません（多分）。いわゆるログの取りこぼしです。
ファイルにも保存しているのはそのためです。ログローテートのタイミングは限られているため、SQL で検索した結果に明らかに矛盾があるとか、時間帯が明らかに怪しいとか、そういう時にはファイルの方を確認します。

多少の取りこぼしを許容するのであれば

```plain:/etc/logrotate.d/mysqld
/var/log/mysql/query.log {
    create 640 mysql mysql
    daily
    rotate 30
    missingok
    sharedscripts
    postrotate
    # just if mysqld is really running
    if test -x /usr/bin/mysqladmin && \
       /usr/bin/mysqladmin ping &>/dev/null
    then
       /usr/bin/mysqladmin flush-logs
       
       /usr/bin/mysql -u root <<_SQL_
SET @general_log_backup = @@GLOBAL.general_log;
SET @@GLOBAL.general_log = 'OFF';

RENAME TABLE mysql.general_log TO mysql.general_log_temp;
DELETE FROM mysql.general_log_temp WHERE event_time < DATE_ADD(NOW(), INTERVAL -30 DAY);
RENAME TABLE mysql.general_log_temp TO mysql.general_log;

SET @@GLOBAL.general_log = @general_log_backup;
_SQL_
    fi
    endscript
}
```

のようにすれば大丈夫だと思います。

上記の SQL だけ取り出したのが下記です。

```sql
SET @general_log_backup = @@GLOBAL.general_log;
SET @@GLOBAL.general_log = 'OFF';

RENAME TABLE mysql.general_log TO mysql.general_log_temp;
DELETE FROM mysql.general_log_temp WHERE event_time < DATE_ADD(NOW(), INTERVAL -30 DAY);
RENAME TABLE mysql.general_log_temp TO mysql.general_log;

SET @@GLOBAL.general_log = @general_log_backup;
```

ログを無効にしないと log tables への ALTER(RENAME) 操作はできないので、前後で無効/有効を切り替えています。
冒頭に記述の通り、general_log への DELETE 操作は行えない（「You can't use locks with log tables」と怒られる）ので、一度リネームし、そいつを DELETE したあと元の名前に戻します。

この場合、ログ自体を無効にしているので取りこぼしもクソもありません。
よってファイルログは不要です。

```ini:/etc/my.cnf
general_log      = 1
general_log_file = /var/log/mysql/query.log
log_output       = TABLE
```
