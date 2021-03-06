---
layout: manual_ja
title: genn.ai
---

## LanguageManual DML

### FROM

FROM は、Tupleを入力先から読み込みます。

#### 入力先が外部の場合

    FROM schema_name AS schema_alias, ... USING spout_processor


* schema_name には、Tuple名もしくはView名を指定します。
* schema_alias には、クエリ内で使用するTupleもしくはViewの別名を指定します。
* spout_processor には、読み込みに使用するプロセッサを指定します。

> Example:
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout()


外部入力は、一つのTopologyに対して一つしか定義できません。

#### Spout Processor

kafka_spout

> TupleをKafkaから読み込みます。システムのデフォルトとして動作します。
>
    kafka_spout()


#### 入力先が内部（ストリーム）の場合

    FROM stream_name[(schema_alias, ...)], ...


* stream_name には、入力先のストリーム名を指定します。

ストリームからすべてのTupleを読み込む場合

> Example:
    FROM s1, s2


ストリームから特定のTupleのみを読み込む場合

> Example:
    FROM s1(ua1, ua2), s2(v1)

---

### INTO

INTO は、ストリームを分岐・合流させます。

    INTO stream_name

* stream_name には、出力するストリーム名を指定します。

> Example:
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1

INTO を使って出力したストリームは、FROM で読み込みます。

#### 分岐

    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1;
    FROM s1(ua1) ...
    FROM s1(ua2) ...
    FROM s1(v1) ...

#### 合流

    FROM s1(ua1) ... INTO s2;
    FROM s1(ua2) ... INTO s3;
    FROM s1(v1) ... INTO s4;
    FROM s2, s3, s4 ...

---

### JOIN

JOINは、外部データをフィールドとしてTupleに結合します。

    JOIN join_name ON join_condition TO join_fields USING fetch_processor

    join_condition:
    join_name.key_field = field [AND join_name.key_field = field AND join_name.key_field <> 0]

    join_fields:
    join_name.join_field AS field_alias, ...


* join_name には、結合する名称を指定します。join_condition や join_fields で、外部データのフィールドを識別する為に使用します。
* join_condition には、結合条件を指定します。
結合データのキーフィールド = Tupleのフィールド を、結合データが一意になるように指定してください。
複合条件の場合は、AND で指定します。結合データのキーフィールドに対して、定数で条件を指定することもできます。
* join_fields には、結合するフィールドを指定します。
結合データのフィールド名 AS Tupleに結合する際のフィールド名 を指定します。フィールドはTupleに追加されます。

> Example:
    JOIN j1 ON j1.code1 = field1 AND j1.code2 = field2 AND j1.del = 0
      TO j1.name AS field10, j1.type AS field11
      USING mongo_fetch('db1', 'col1')

#### EXPIRE

```
JOIN join_name ON join_condition TO join_fields EXPIRE expire_period
USING fetch_processor
```

TO 〜 の 後ろに、「EXPIRE キャッシュの有効期限」で指定します。
EXPIRE 〜 を省略した場合は、キャッシュは実行されません。

指定した時間の間、fetchした内容をキャッシュします。キャッシュしている間に実行されたJOINは、キャッシュから結合フィールドを取得します。
指定した時間が過ぎると、ふたたびFetchProcessorを実行してキャッシュを更新します。

キャッシュは結合キーごとに保存されます。

> Example:
> JOIN books ON books.title = ccc AND books.author = ddd TO books.id AS book_id, books.price AS price EXPIRE 10min
> USING web_fetch('http://localhost:3000/solr/select?q=${query}', [' = ',':', ' AND ', '+AND+'], 'response.docs[0]')

---

### FILTER

FILTER は、単一のTupleに対してTupleの通過を判定します。

    FILTER condition

* condition には、フィルタの条件を指定します。

 condition の符号には、以下のものを指定します。

* &#61; もしくは &#61;&#61;
* <> もしくは !&#61;
* &gt;
* &gt;=
* &lt;
* &lt;=
* LIKE
* REGEXP
* IN
* ALL
* BETWEEN
* AND
* OR
* NOT

#### &#61;, &#61;&#61;, <>, !&#61;, >, >&#61;, <, <&#61;

> Example:
    field1 >= 10

#### LIKE

LIKEで使用できるワイルドカードは、"%"（複数文字）と"_"（一文字）です。

> Example:
    field2 LIKE 't%'

#### REGEXP

REGEXPで使用できる正規表現は、java/util/regex/Patternと同じ書式を採用しています。
http://docs.oracle.com/javase/6/docs/api/java/util/regex/Pattern.html

> Example:

    field3 REGEXP '^[A-Z]{2}-[0-9]{4}$'

#### IN, ALL

INは、LISTフィールドに対して値が一つでも含まれているかを調べます。

> Example:

    field4 IN ('tokyo', 'kyoto', 'osaka')

ALLは、LISTフィールドに対して値がすべて含まれているかを調べます。

> Example:

    field4 ALL ('tokyo', 'kyoto', 'osaka')

#### BETWEEN

> Example:

    field1 10 AND 100


#### AND, OR, NOT

AND, OR, NOT は入れ子にすることが可能です。優先順位はNOT, AND, ORの順に処理されます。
優先順位は、()を使用して変更できます。

> Example:
    field1 <= 30 AND (field5 BETWEEN 10 AND 100 OR field2 LIKE 'A%')

#### STRUCT型フィールドの比較

TupleのフィールドがSTRUCT型の場合は、フィールド値を以下のように比較します。

> Example:
    field6.member3 = 100

#### LIST型フィールドの比較

TupleのフィールドがLIST型の場合は、フィールド値を以下のように比較します。

> Example:
    field4[0] = 'tokyo'

#### MAP型フィールドの比較

TupleのフィールドがMAP型の場合は、フィールドの値を以下のように比較します。

> Example:
    field7['visa'] = true

#### 定数

条件に指定できる定数は、以下になります。

* 文字列
シングルクォートまたはダブルクォートでくくった文字列

* INT値
    number only

>
> Example:
    2147483647


* DOUBLE値
    number.number

>
> Example:
    12.5


* BIGINT値
    numberL

>
> Example:
    9223372036854775807L


* SMALLINT値
    numberS

>
> Example:
    32767S


* TINYINT値
    numberY

>
> Example:
    255Y


* FLOAT値
    number.numberF

>
> Example:
    12.5F


* BOOLEAN値
    true|false


---

### FILTER GROUP

FILTER GROUP は、複数のTupleに対してTupleの通過を判定します。

    FILTER GROUP EXPIRE period [STATE TO state_field]
      condition, ...


* period には、フィルタの状態を保持する期間を指定します。
* state_field には、フィルタの状態を出力するフィールド名を指定します。フィルタを通過すると、指定したフィールド名で
Tupleに状態フィールドを追加します。STATE TO clause を省略した場合は、状態フィールドは追加しません。
* condition には、フィルタ条件を指定します。カンマ区切りで、複数のTupleに対する条件を指定します。
カンマ区切りで指定した条件をすべて満たせば、Tupleはフィルタを通過します。
すべての条件を満たした場合、最後に到着したTupleがフィルタを通過し、フィルタの状態は初期化されます。

> Example:
    FILTER GROUP EXPIRE 7DAYS STATE TO fg_state
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member2 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'


 ua1, ua2, ua3の３つのTupleに対してフィルタを実行します。条件１と条件２（※）の両方を満たせば、Tupleはフィルタを通過します。
 （※）条件１はua1に対するフィルタで、条件２はua2とua3に対するフィルタです。

 条件２は、ua2とua3の条件を"OR"で指定しているので、ua1かつua2の条件を満たすか、ua1かつua3の条件を満たすことで
 すべての条件は満たされます。

 条件１もしくは条件２のいずれかを満たした状態を７日間保持します。
 例えば、条件１を最後に満たした状態で、条件２の条件を満たすまでの期間が７日以内であれば、条件１と条件２は満たされますが、
 ８日以上の日数が経過していた場合、条件１の状態は初期化されてしまう為、条件２のみ満たしている状態になります。

 state_fieldに"fg_state"を指定しているので、フィルタの状態を"fg_state"フィールドとしてTupleに追加します。
 fg_stateは、条件１と条件２のそれぞれを満たした日時が格納されます。条件の数と等しいTIMESTAMPのLISTになります。

#### period

* 秒で指定
    number(SECONDS|SEC)
>
> Example:
    30SECONDS


* 分で指定
    number(MINUTES|MIN)
>
> Example:
    55MIN

* 時間で指定
    number(HOURS|H)
>
> Example:
    55HOURS


* 日で指定
    number(DAYS|D)

> Example:
    15DAYS

---

### EACH

EACH は、Tupleの集計や編集を実行します。

  EACH expr, ...

* exprには、集計関数もしくは編集関数、フィールドのアクセサを指定します。

#### 集計関数

* Tupleの到着数をカウントします。
>
> Example:
    EACH count() AS cnt1


* 到着したフィールドの値を合計します。
>
> Example:
    EACH sum(field1) AS sum1


* 到着したフィールドの値の平均を計算します。
>
> Example:
    EACH avg(field1) AS avg1


#### 編集関数

* フィールドの値がNULLであれば、代替えの値で置き換えます。
>
> Example:
    EACH ifnull(field1, 0) AS field1


* STRING型のフィールドの値を連結したフィールドを作成します。
>
> Example:
    EACH concat(field1, '-', field2) AS new_field


#### 関数の引数

定数に関しては、関数の種類が増えてきてから改めて記述する。

#### フィールドのアクセサ

> Example:
    EACH field1, field6.member1 AS field10, field7['visa'] AS visa

field1はそのまま、field6.member1をfield10フィールドへ、field7&#91;'visa']をvisaフィールドへ抽出します。

---

### SLIDE

集計関数によるスライド集計できます。

```
SLIDE LENGTH [period BY time_field| count] 集計関数
```

#### 時間でスライド

```
SLIDE LENGTH 10sec BY _time sum(bbb) AS s, count() AS c
```

LENGTH スライド時間 BY 起点となるフィールド名 集計関数
といった書式で指定します。

* 起点となるフィールド名には、Timestamp型のフィールドを指定します。
  _timeフィールドも指定できます。（スキーマに_timeフィールドを指定していた場合）
* 集計関数は複数指定できます。現状では、sum(), avg(), count()の３つの集計関数が使用できます。

#### 件数でスライド

```
SLIDE LENGTH 100 sum(bbb) AS s, count() AS c
```

LENGTHにスライドするTupleの数を指定します。

SLIDE句はSlideOperatorを生成して処理されます。
スライドに使用する領域は現在メモリになっていますが、将来的に外部DBへの差し替えを可能にしたいと考えています。
スライドはTupleの到着時にのみ実行されます。（スライドによる再計算も到着時のみ）

SlideOperatorは、Tupleが到着する度にTupleをウィンドウ領域に貯めつつ、都度集計を行なっていきます。
その際、ウィンドウ幅から除外されたTuple（※）を集計結果から減算します。
（※）スライドして集計対象から外れたTuple

ウィンドウ領域に保存するTupleは、集計に必要なフィールドのみを選択しています。

---

### SNAPSHOT

指定された期間もしくは、時間、もしくは件数毎に集計を実施します。

#### 期間毎に集計

例えば、7分毎に集計結果を返すとします。

```
SNAPSHOT EVERY 7min sum(bbb) AS s, count() AS c
SNAPSHOT EVERY "*/7 * * * *" sum(bbb) AS s, count() AS c
```

集計はTupleが送られてくる度に実行されますが、集計結果は７分に１回だけ次のOperatorに送られます。
下段の例は、時間をcron形式で指定したものです。（注：ダブルクォートでくくる必要があります）
７分の間に１度もTupleが送られてこなかった場合は、集計結果が生成されない為、Tupleは次のOperatorに送られません。
指定できる時間は１分以上です。
集計結果は７分毎にリセットされます。

#### 指定した時間に集計

例えば、毎日0時に日時の集計結果を返すとします。

```
SNAPSHOT EVERY "0 0 * * *" sum(bbb) AS s, count() AS c
```

基本はcronと同じ書式です。範囲指定、カンマ区切りによる複数指定が可能です。

#### 定数毎に集計

例えば、Tupleを10個毎に集計した結果を返すとします。

```
SNAPSHOT EVERY 10 sum(bbb) AS s, count() AS c
```

指定した回数分のTupleが届いたタイミングで、集計結果が次のOperatorに送られます。
集計結果は10件毎にリセットされます。

---

### GROUP

BEGIN GROUP ... END GROUP で囲まれたクエリを、グループで実行します。

    BEGIN GROUP BY field, ...
      [END GROUP | TO STREAM]

* field には、グループ化するフィールドの名前を指定します。

#### EACH をグループで実行する

> Example:
    BEGIN GROUP BY user_name
    EACH user_name, count() AS gc1
    EMIT * USING mongo_persist('db1', 'col2', 'user_name');
    END GROUP


user_name ごとに（ユーザごとに）Tupleがカウントされます。

#### FILTER GROUP をグループで実行する

> Example:
    BEGIN GROUP BY user_name
    FILTER GROUP EXPIRE 1DAYS
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member3 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'
    EMIT * USING mongo_persist('db1', 'col3')
    END GROUP


user_nameごとに（ユーザごとに）フィルタが判定されます。
特定のユーザが条件１と条件２を満たしているかを判定し、FILTER GROUP の状態はユーザごとに保持されます。

#### GROUP のネスト

GROUPはネストできます。

> Example:
    EACH ...  <- グループ化せずに実行
    BEGIN GROUP BY date
      EACH ...  <- date ごとに実行される
      BEGIN GROUP BY area
        EACH ...  <- date + area ごとに実行される
      END GROUP
    END GROUP
    EACH ...  <- グループ化せずに実行


TO STREAM で、すべてのグループを解除します。

> Example:
    EACH ...  <- グループ化せずに実行
    BEGIN GROUP BY date
      EACH ...  <- date ごとに実行される
      BEGIN GROUP BY area
        EACH ...  <- date + area ごとに実行される
    TO STREAM
    EACH ...  <- グループ化せずに実行


END GROUP と TO STREAM は、グループ化を解除する必要がなければ省略可能です。

---

### LIMIT

Tupleの流れを制限したい場合に使用します。

```
LIMIT FIRST|LAST EVERY condition

condition:
period | number
```

#### 時間による制限

* 最初に届いたTupleのみを次のOperatorに送り、以降30分の間はTupleを送りません。

```
LIMIT FIRST EVERY 30min
```

* Tupleが最初に届いてから、30分の間に届いたTupleの最後の１件のみを次のOperatorに送ります。

```
LIMIT LAST EVERY 30min
```

* 時間の起点は、最初にTupleが届いた時になります。起点は30分経過後にリセットされます。
* 30分経過したかどうかは、30分経過後に初めてTupleが届いた時点で判断されます。　　
LASTを指定した場合で、30分間の最後にTupleが届いた時点から、次のTupleが届くまでに１週間かかった場合は、最後のTupleが送られるのは１週間後になります。

#### 件数による制限

* ５件のTuple中で、最初に届いたTupleを通します。

```
LIMIT FIRST EVERY 5
```

4. ５件のTuple中で、最後に届いたTupleを通します。

```
LIMIT LAST EVERY 5
```

 * いずれもカウンタは５件ごとにクリアされます。

#### 補足

LIMITを使うことによって、
LIMIT FIRSTであれば、１度EMIT（通知）したTupleを一定時間EMITさせないように制限したり、
LIMIT LASTであれば、最初にアクションし始めてから、４時間の間で一番最後に行ったアクションを抽出したりすることができると思います。

LIMIT LASTの場合、Tupleの通過のトリガーが、時間経過後に初めてきたTupleになるので、通過までタイムラグが発生します。これを時間経過と同時に通過させる
（時間をトリガーにする）ことも可能ですが、どちらがいいのか迷いました。。

---

### EMIT

EMITは、Tupleを外部へ出力します。

  EMIT output_field, ... USING emit_processor

* output_field には、出力するフィールド名を指定します。ワイルドカード（&#42;）を指定できます。
* emit_processor には、出力に使用するプロセッサを指定します。

> Example:
    EMIT field1, field2, field3 USING mongo_persist('db1', 'col1')

#### Emit Processor

* Kafka Emit Processor  

TupleをKafkaに出力します。

    kafka_emit(topic_name)


 * topic_name には、出力するTopic名を指定します。topic_name はプロセッサ変数に対応しています。

> Example:

    kafka_emit('topic1')


* Mongo Persist Processor

TupleをMongoDBに出力します。

    mongo_persist(db_name, collection_name [, key_names])


* db_name には、出力するDB名を指定します。db_name はプロセッサ変数に対応しています。
* collection_name には、出力するCollection名を指定します。collection_name はプロセッサ変数に対応しています。
* key_names には、出力するキーのフィールド名を指定します。複合キーの場合は配列で指定してください。
 key_names を指定した場合、出力はキーに対してupdateされます。
 key_names を指定しなかった場合は、出力はinsertになります。

> Example:
    mongo_persist('db1', 'col1')  <- insert
    mongo_persist('db1', 'col1', 'field2') <- field2 をキーとしてupdate
    mongo_persist('db1', 'col1', ['field2', 'field3']) <- field2 + field3 を複合キーとしてupdate


#### プロセッサ変数

Emit Processor の出力先の名称には、以下のプロセッサ変数を含めることができます。

${TOPOLOGY_ID} は、起動中のTopology IDに置き換えられます。
${ACCOUNT_ID} は、Topologyを起動したユーザのAccount IDに置き換えられます。

> Example:
    kafka_emit('topic_${TOPOLOGY_ID}')
