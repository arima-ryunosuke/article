# mysql の 1レコード 8KB の壁を確認するやつ


ちょっと古めの環境でかつ設定をいじれない系サーバでの開発があって、8KB 制限が怖いので確認のため書いた。

## コード

```php
function calculateColumnSize(\PDO $pdo, $table)
{
    $result = array();

    $columns = $pdo->query("DESCRIBE $table")->fetchAll(\PDO::FETCH_ASSOC);
    foreach ($columns as $column) {
        $name = $column['Field'];
        $type = $column['Type'];

        switch (true) {
            // 整数系
            case preg_match('#^tinyint#', $type):
                $result[$name] = 1;
                break;
            case preg_match('#^smallint#', $type):
                $result[$name] = 2;
                break;
            case preg_match('#^mediumint#', $type):
                $result[$name] = 3;
                break;
            case preg_match('#^int#', $type):
                $result[$name] = 4;
                break;
            case preg_match('#^bigint#', $type):
                $result[$name] = 8;
                break;

            // 浮動小数系
            case preg_match('#^float$#', $type):
                $result[$name] = 4;
                break;
            case preg_match('#^double$#', $type):
                $result[$name] = 8;
                break;

            // 固定小数系
            case preg_match('#^decimal\((\d+),(\d+)\)$#', $type, $match):
                // 整数部と小数部で計算式は同じ(9の商*4 byte, 余 / 2 byte)
                $digits = array($match[1] - $match[2], $match[2]);
                $result[$name] = 0;
                foreach ($digits as $digit) {
                    $result[$name] += floor($digit / 9) * 4;
                    $result[$name] += ceil($digit % 9 / 2);
                }
                break;

            // 文字列/バイナリ系
            case preg_match('#^(var)?char\((\d+)\)$#', $type, $match):
            case preg_match('#^(var)?binary\((\d+)\)$#', $type, $match):
                // utf8 なら *3 byte。ただし先頭 768 byte まで…のはず
                // todo varchar だと文字長格納で数バイト使うんだっけか？
                $result[$name] = min(768, $match[2] * 3);
                break;

            // TEXT/BLOB 系
            case preg_match('#^(tiny|medium|long)?text$#', $type):
            case preg_match('#^(tiny|medium|long)?blob$#', $type):
                // 最大値で見積もる
                $result[$name] = 256 * 3;
                break;

            // 日時系
            case preg_match('#^datetime$#', $type):
                $result[$name] = 8;
                break;
            case preg_match('#^date$#', $type):
                $result[$name] = 3;
                break;
            case preg_match('#^time$#', $type):
                $result[$name] = 3;
                break;
            case preg_match('#^year#', $type):
                $result[$name] = 1;
                break;
            case preg_match('#^timestamp$#', $type):
                $result[$name] = 4;
                break;

            // ENUM
            case preg_match('#^enum\((.+)\)$#', $type, $match):
                // 列挙値が 255 までは 1byte, それ以上は 2byte
                $count = count(str_getcsv($match[1], ',', "'", "\\"));
                $result[$name] = $count <= 255 ? 1 : 2;
                break;

            // BIT とか SET とか
            default:
                $result[$name] = null;
                break;
        }
    }

    return $result;
}
```

※ utf8 前提
※ BIT とか GEOMETRY は使ってないので null ぶっこみ
※ というか文字列系がでかすぎて誤差レベル

## 確認

```sql
CREATE TABLE t_hogera (
    c_tinyint TINYINT(4) NULL DEFAULT NULL,
    c_smallint SMALLINT(6) NULL DEFAULT NULL,
    c_mediumint MEDIUMINT(9) NULL DEFAULT NULL,
    c_int INT(11) NULL DEFAULT NULL,
    c_bigint BIGINT(20) NULL DEFAULT NULL,
    c_float FLOAT NULL DEFAULT NULL,
    c_double DOUBLE NULL DEFAULT NULL,
    c_decimal0602 DECIMAL(6,2) NULL DEFAULT NULL,
    c_decimal3030 DECIMAL(30,30) NULL DEFAULT NULL,
    c_char32 CHAR(32) NULL DEFAULT NULL COLLATE 'utf8_bin',
    c_char128 CHAR(128) NULL DEFAULT NULL COLLATE 'utf8_bin',
    c_varchar32 VARCHAR(32) NULL DEFAULT NULL COLLATE 'utf8_bin',
    c_varchar128 VARCHAR(128) NULL DEFAULT NULL COLLATE 'utf8_bin',
    c_tinytext TINYTEXT NULL COLLATE 'utf8_bin',
    c_text TEXT NULL COLLATE 'utf8_bin',
    c_mediumtext MEDIUMTEXT NULL COLLATE 'utf8_bin',
    c_longtext LONGTEXT NULL COLLATE 'utf8_bin',
    c_binary32 BINARY(32) NULL DEFAULT NULL,
    c_binary128 BINARY(128) NULL DEFAULT NULL,
    c_varbinary32 VARBINARY(32) NULL DEFAULT NULL,
    c_varbinary128 VARBINARY(128) NULL DEFAULT NULL,
    c_tinyblob TINYBLOB NULL,
    c_blob BLOB NULL,
    c_mediumblob MEDIUMBLOB NULL,
    c_longblob LONGBLOB NULL,
    c_date DATE NULL DEFAULT NULL,
    c_time TIME NULL DEFAULT NULL,
    c_year YEAR NULL DEFAULT NULL,
    c_datetime DATETIME NULL DEFAULT NULL,
    c_timestamp TIMESTAMP NULL DEFAULT NULL,
    c_enum ENUM('OK','NG') NULL DEFAULT NULL COLLATE 'utf8_bin'
);
```

このようなテーブルで

```php
$pdo = new \PDO('mysql:host=localhost;dbname=test', 'ore', 'are');
$sizes = calculateColumnSize($pdo, 't_hogera');

print_r($sizes);
echo 'total ', number_format(array_sum($sizes)), ' bytes.';
```

を実行すると

```
Array
(
    [c_tinyint] => 1
    [c_smallint] => 2
    [c_mediumint] => 3
    [c_int] => 4
    [c_bigint] => 8
    [c_float] => 4
    [c_double] => 8
    [c_decimal0602] => 3
    [c_decimal3030] => 14
    [c_char32] => 96
    [c_char128] => 384
    [c_varchar32] => 96
    [c_varchar128] => 384
    [c_tinytext] => 768
    [c_text] => 768
    [c_mediumtext] => 768
    [c_longtext] => 768
    [c_binary32] => 96
    [c_binary128] => 384
    [c_varbinary32] => 96
    [c_varbinary128] => 384
    [c_tinyblob] => 768
    [c_blob] => 768
    [c_mediumblob] => 768
    [c_longblob] => 768
    [c_date] => 3
    [c_time] => 3
    [c_year] => 1
    [c_datetime] => 8
    [c_timestamp] => 4
    [c_enum] => 1
)
total 8,131 bytes.
```

って出てくる。
うん、いい感じ。

全く関係ないんだけど、

- switch (true)
- キャプチャ付き `preg_match` を並列に並べる

の合わせ技って普通に動作するのね…。
