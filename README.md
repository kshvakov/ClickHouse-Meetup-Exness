# ClickHouse-Meetup-Exness 07.05.2019

# Стратегии оптимизации запросов и доступа к данным в ClickHouse

ClickHouse не тормозит ©. Но это не точно.

## Традиционно.

Что влияет на скорость выполнения запросов?

* CPU
* Память
* Сеть
* "RAID с батарейкой"
* "Хороший RAID с батарейкой"
* Попутный ветер

Это все не важно.

* чем больше - тем лучше
* чем лучше  - тем лучше

## Менее традиционно.

если хочется улучшить - нужно переписывать код :)

* https://github.com/littledan/linux-aio
* https://blog.cloudflare.com/io_submit-the-epoll-alternative-youve-never-heard-about/
* https://blog.cloudflare.com/sockmap-tcp-splicing-of-the-future/
* https://github.com/search?l=C%2B%2B&q=fast&type=Repositories
* https://github.com/search?l=C%2B%2B&q=High-performance&type=Repositories

Важно, но сложно!

## Есть теория.

> Если какие-то вычисления занимают X секунд, то для того, чтобы нам потребовалось X/2 секунд, нужно вычислять X/2.
<p style='text-align: right;'>теория и практика эффективного менеджмента Т.П. Барсук. МСК 2003.</p>

> Существует мнение, что для того, чтобы эффективно считать статистику, данные нужно агрегировать, так как это позволяет уменьшить объём данных.
<p style='text-align: right;'> выдержка из документации к ClickHouse</p>


> И это правда :)


## Агрегировать?

"Сырые" события.

В Facebook на этот митап "собирались пойти" или "интересовались" 250 человек.
Представим, что каждый раз когда человек нажимал на кнопку FB отправлял нам данные:

* тип ("пойду" или "интересно")
* время с точностью до секунды
* имя
* пол
* день рождения
* место рождения
* место работы
* должность

Мы записали данные как есть - это и будут "сырые" данные.

```sql
CREATE TABLE exness_events (
      event_time    DateTime
    , event_type    Enum16('going' = 1, 'interested' = 2)
    , name          String
    , gender        Enum16('male'  = 1, 'female'     = 2)
    , date_of_birth Date
    , birthplace    String
    , company       String
    , position      String
) Engine MergeTree PARTITION BY toYYYYMM(event_time) ORDER BY (event_type, event_time);
```

Зачем вообще заранее агрегировать?

Нам нужно понимать сколько человек собирается посетить мероприятие, для этого нам нужно знать общее количество человек.
Для этого нам совершенно не нужно каждый раз проверять все записи, достаточно создать таблицу в которую мы запишем только тип события и их количество.

```sql
CREATE TABLE exness_events_agg (
    event_type    Enum16('going' = 1, 'interested' = 2)
    , total       Int16
) Engine SummingMergeTree PARTITION BY event_type ORDER BY (event_type);

-- создадим триггер который будет вызываться при каждой записи в exness_events и писать в exness_events_agg
CREATE MATERIALIZED VIEW exness_events_mv TO exness_events_agg AS
    SELECT event_type, toInt16(1) AS total FROM exness_events;
```

Давайте проверим

```sql
INSERT INTO exness_events (
    event_time
    , event_type
    , name
    , gender
    , date_of_birth
    , birthplace
    , company
    , position
) VALUES
      ('2019-05-01 12:15:41', 'going',      'Kirill',   'male',   '1983-12-13', 'Russia',  'Integros Video Platform', 'Golang developer')
    , ('2019-05-01 12:15:41', 'going',      'Mister V', 'male',   '1983-01-01', 'Russia',  'Aloha Browser',           'Backend developer')
    , ('2019-05-02 08:12:11', 'going',      'Miss X',   'female', '1995-10-25', 'Russia',  'Company A',               'T')
    , ('2019-05-02 08:12:11', 'going',      'Miss Y',   'female', '1985-01-24', 'Ukraine', 'Company A',               'T')
    , ('2019-05-02 11:15:11', 'interested', 'Mister Y', 'male',   '1983-01-01', 'Ukraine', 'Company B',               'T');

OPTIMIZE TABLE exness_events_agg FINAL;

SELECT * FROM exness_events_agg;

┌─event_type─┬─total─┐
│ going      │     4 │
│ interested │     1 │
└────────────┴───────┘
```

Все работает ожидаемым образом вместо X записей у нас всегда будет достаточно небольшое количество в арегированных (заранее просуммированных) значений.

ClickHouse в фоне производит агрегацию в `SummingMergeTree` (вся "магия" MergeTree происходит во время слияний), поэтому мы не можем быть уверенными что в момент времени T все данные уже просумировались и правильный запрос будет выглядеть так:

```sql
SELECT
    event_type
    , SUM(total) AS total
FROM exness_events_agg
GROUP BY event_type
```

Однако, если нам вдруг потребудется еще один статистический срез, то нам потребуется либо создавать еще одну таблицу, либо добавлять колонку к уже существующей таблице `exness_events_agg`.

Если нам потребуется много отчетов, то если пользоваться первым советом и создавать таблицы, то в какой-то момент их может стать слишком много, если пользоваться советом N 2, то при наличии достаточно юольшого количества колонок и различных вариантов агрегация может стать бессмысленной.

Например если мы захотим агрегировать данные по колонкам: event_time, event_type, name, gender и date_of_birth то мы получим количество данных которое будет очень близко к сырым.

## Что делать?


Действительно, представим себе, что нам нужны отчеты с разбивкой по полу, типу события и дате:

* какие имена
* возраст на момент регистрации
* в какой компании работет
* должности


Для этого пересоздадим таблицу `exness_events_agg` и материализованное представление `exness_events_mv`

```sql
TRUNCATE TABLE exness_events;
DROP TABLE exness_events_agg;
DROP TABLE exness_events_mv;

CREATE TABLE exness_events_agg (
      event_date Date
    , event_type Enum16('going' = 1, 'interested' = 2)
    , gender     Enum16('male'  = 1, 'female'     = 2)
    , total      Int64
    , nameMap    Nested(
          id    UInt64
        , total Int64
    )
    , ageMap    Nested(
          age   Int8
        , total Int64
    )
    , companyMap Nested(
          id    UInt64
        , total Int64
    )
    , positionMap Nested(
          id    UInt64
        , total Int64
    )
) Engine SummingMergeTree PARTITION BY toYYYYMM(event_date) ORDER BY (
    event_type
    , gender
    , event_date
);

CREATE MATERIALIZED VIEW exness_events_mv TO exness_events_agg AS
    SELECT
          toDate(event_time) AS event_date
        , event_type
        , gender
        , toInt64(1)                                AS total
        , [cityHash64(name)]                        AS `nameMap.id`
        , [total]                                   AS `nameMap.total`
        , [toInt8((today() - date_of_birth) / 365)] AS `ageMap.age`
        , [total]                                   AS `ageMap.total`
        , [cityHash64(company)]                     AS `companyMap.id`
        , [total]                                   AS `companyMap.total`
        , [cityHash64(position)]                    AS `positionMap.id`
        , [total]                                   AS `positionMap.total`
    FROM exness_events;
```

К сожалению на данным момент ни SummingMergreTree, ни функция sumMap не могут складывать оперировать строками в качестве агрументов, поэтому чтоб суммирование одинаковых значений мы считаем хэш от строки и сохраняем его.

Так как мы храним хэшированое значение, то нам нужно будет его преобразовывать обратно, для этого создадим словарь и сохраним в нем пару ключ-значение.

```sql
CREATE TABLE exness_dictionary (
    id UInt64
    , value String
) Engine ReplacingMergeTree
PARTITION BY tuple()
ORDER     BY id;

CREATE MATERIALIZED VIEW exness_dictionary_mv TO exness_dictionary AS
    SELECT DISTINCT
          cityHash64(value) AS id
        , value
    FROM exness_events
    ARRAY JOIN
    [
        name
        , company
        , position
    ] AS value
;
```

Пример файла конфигурации словаря для ClickHouse.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<dictionaries>
    <dictionary>
        <name>exness</name>
        <source>
            <clickhouse>
                <host>127.0.0.1</host>
                <port>9000</port>
                <user>default</user>
                <password />
                <db>default</db>
                <table>exness_dictionary</table>
            </clickhouse>
        </source>
        <lifetime>600</lifetime>
        <layout>
            <hashed />
        </layout>
        <structure>
            <id>
                <name>id</name>
            </id>
            <attribute>
                <name>value</name>
                <type>String</type>
                <null_value />
            </attribute>
        </structure>
    </dictionary>
</dictionaries>
```

снова запишем наши данные

```sql
INSERT INTO exness_events (
    event_time
    , event_type
    , name
    , gender
    , date_of_birth
    , birthplace
    , company
    , position
) VALUES
      ('2019-05-01 12:15:41', 'going',      'Kirill',   'male',   '1983-12-13', 'Russia',  'Integros Video Platform', 'Golang developer')
    , ('2019-05-01 12:15:41', 'going',      'Mister V', 'male',   '1983-01-01', 'Russia',  'Aloha Browser',           'Backend developer')
    , ('2019-05-02 08:12:11', 'going',      'Miss X',   'female', '1995-10-25', 'Russia',  'Company A',               'T')
    , ('2019-05-02 08:12:11', 'going',      'Miss Y',   'female', '1985-01-24', 'Ukraine', 'Company A',               'T')
    , ('2019-05-02 11:15:11', 'interested', 'Mister Y', 'male',   '1983-01-01', 'Ukraine', 'Company B',               'T');

OPTIMIZE TABLE exness_events_agg FINAL;

SELECT *
FROM exness_events_agg

┌─event_date─┬─event_type─┬─gender─┬─total─┬─nameMap.id──────────────────────────────────┬─nameMap.total─┬─ageMap.age─┬─ageMap.total─┬─companyMap.id──────────────────────────────┬─companyMap.total─┬─positionMap.id──────────────────────────────┬─positionMap.total─┐
│ 2019-05-01 │ going      │ male   │     2 │ [2755368670248224592,14035790109297986913]  │ [1,1]         │ [35,36]    │ [1,1]        │ [4981400259713581340,14222025201026224911] │ [1,1]            │ [13769500187843876357,13801179002127203999] │ [1,1]             │
│ 2019-05-02 │ going      │ female │     2 │ [11150560953386856707,11247670348204915763] │ [1,1]         │ [23,34]    │ [1,1]        │ [16119125104385189742]                     │ [2]              │ [5570910369516329963]                       │ [2]               │
│ 2019-05-02 │ interested │ male   │     1 │ [2871166117508672652]                       │ [1]           │ [36]       │ [1]          │ [7418537249491522274]                      │ [1]              │ [5570910369516329963]                       │ [1]               │
└────────────┴────────────┴────────┴───────┴─────────────────────────────────────────────┴───────────────┴────────────┴──────────────┴────────────────────────────────────────────┴──────────────────┴─────────────────────────────────────────────┴───────────────────┘
```

И так, мы видем что мы можем построить наш базовый отчет

```sql
SELECT
    event_type
    , SUM(total) AS total
FROM exness_events_agg
GROUP BY event_type

┌─event_type─┬─total─┐
│ going      │     4 │
│ interested │     1 │
└────────────┴───────┘
```

Но теперь у нас гораздо больше данных и мы можем создавать и дргие отчеты:

```sql
-- нам интересно сколько девушек собиается пойти
SELECT
    SUM(total) AS going
FROM exness_events_agg
WHERE event_type = 'going' AND gender = 'female';

┌─going─┐
│     2 │
└───────┘

-- возраст :)

SELECT
      ageMap.age AS age
    , SUM(ageMap.total) AS going
FROM exness_events_agg ARRAY JOIN ageMap
WHERE event_type = 'going' AND gender = 'female'
GROUP BY age;

┌─age─┬─going─┐
│  23 │     1 │
│  34 │     1 │
└─────┴───────┘

-- может быть в каких компаниях работают?

SELECT
      dictGetString('exness', 'value', companyMap.id)            AS company
    , SUM(companyMap.total) AS going
FROM exness_events_agg ARRAY JOIN companyMap
WHERE event_type = 'going' AND gender = 'female'
GROUP BY company;

┌─company───┬─going─┐
│ Company A │     2 │
└───────────┴───────┘
```

Как мы видем даных более чем достаточно для построения различных отчетов, при этом строк в агрегированной таблице будет значительно меньше чем в таблице с исходными данными.

Однако, если посмотреть на структуру станет очевидным то, что мы, несмотря на то что все данные у нас остались сохранены, мы из-за агрегации потеряли часть связей и не можем построить запрос, например, который выдаст нам кто в какой компании и на какой должности работает.

> В качестве вывода. Иногда агрегация может сильно выручать сильно уменьшая объем данных, однако не стоит забывать, что в этом случае мы жертвуем как минимум потерей части связей между данными, что может быть как плохо, так и нет (например если вам достаточно заранее подготовленных отчетов).

* я знаю, вывод какая-то фигня, его надо будет переписать :)

## Часть вторая, когда просто пописать SQL может быть недостаточно.

Давайте немного усложним задачу.

Представим что у нас есть некий сервис, например это некоторая видеоплатформа.
На платформе работают пользователи, их юзкейс очень простой: пользователи загружают какое-то видео, это видео показывается и нужно видеть статистику по показам и по трафику, а также лог доступа к видео.

Создадим таблицу с сырыми событиями

```sql
CREATE TABLE video_events (
    event_time DateTime
    , user_id  Int32
    , video_id Int64
    , bytes    Int64
    , os       LowCardinality(String)
    , device   LowCardinality(String)
    , browser  LowCardinality(String)
    , country  LowCardinality(String)
    , domain   LowCardinality(String)
) Engine MergeTree PARTITION BY toYYYYMM(event_time) ORDER BY (
    event_time
    , user_id
    , video_id
);
```

И перед тем как действовать дальше прило время уточнить требования.

На самом деле статистику должен видеть владелец как по всем видео так и с детализацией по конкретному видео, причем нам важны просмотры по датам.
Также в системе есть администраторы - которые хотят видеть статистику глобально по всем пользователям, а также с детализацией до конкретного пользователя и видео.
Помимо статистики необходимо видеть действия (лог доступа) которые являются ничем иным как сырыми событиями, при этом должна быть фильтрация по всем колонкам.


Мы уже знаем о том, что агрегирование данных может быть хорошим решением.
Исходя из опыта и имеющихся данных мы знаем, что у нас достаточно много событий которые неплохо "свернутся" при агрегации.

Создадим таблицу

```sql
CREATE TABLE video_events_agg (
    date          Date
    , user_id     Int32
    , video_id    Int64
    , os          LowCardinality(String)
    , device      LowCardinality(String)
    , browser     LowCardinality(String)
    , country     LowCardinality(String)
    , domain      LowCardinality(String)
    , bytes_total Int64
    , count       Int64
    , min_time    AggregateFunction(min, DateTime)
    , max_time    AggregateFunction(max, DateTime)
    , timeMap     Nested (
        hour    UInt8
        , count Int64
    )
) Engine SummingMergeTree
PARTITION BY toYYYYMM(date)
PRIMARY KEY (
    date
    , user_id
    , video_id
)
ORDER BY (
    date
    , user_id
    , video_id
    , os
    , device
    , browser
    , country
    , domain
);
-- Как мы видим, наша таблица для агрегированных данных содержит теже колонки что и таблица сырых данных, поэтому мы сможем строить произвольные запросы к ней

CREATE MATERIALIZED VIEW video_events_mv TO video_events_agg AS
    SELECT
        toDate(event_time) AS date
        , user_id
        , video_id
        , os
        , device
        , browser
        , country
        , domain
        , toInt64(1)                            AS count
        , bytes                                 AS bytes_total
        , arrayReduce('minState', [event_time]) AS min_time
        , arrayReduce('maxState', [event_time]) AS max_time
        , [toHour(event_time)]                  AS `timeMap.hour`
        , [count]                               AS `timeMap.count`
    FROM video_events;
```

Запишем немного данных

```sql
INSERT INTO video_events (
    event_time
    , user_id
    , video_id
    , bytes
    , os
    , device
    , browser
    , country
    , domain
)
WITH (
    ['Linux',  'OS X',    'Windows']                    AS oses
  , ['Desktop', 'Mobile', 'Tablet']                     AS devices
  , ['Chrome', 'Firefox', 'IE']                         AS browsers
  , ['CY', 'RU', 'UA', 'BY', 'US', 'UK', 'NL']          AS countries
  , ['clickhouse.yandex', 'altinity.com', 'exness.com'] AS domains
)
SELECT
      now() - number AS event_time
    , U                          AS user_id
    , U * 1000 + (rand() % 1000) AS video_id
    , rand()                     AS bytes
    , oses[(rand()      + user_id * 1) % length(oses)      + 1] AS os
    , devices[(rand()   + user_id * 2) % length(devices)   + 1] AS device
    , browsers[(rand()  + user_id * 3) % length(browsers)  + 1] AS browser
    , countries[(rand() + user_id * 4) % length(countries) + 1] AS country
    , domains[(rand()   + user_id * 5) % length(domains)   + 1] AS domain
FROM numbers(3600*24*30) ARRAY JOIN range(50) AS U, range(5) AS V WHERE U > 0;

-- добавим больше неуникальных событий

INSERT INTO video_events SELECT * FROM video_events;
INSERT INTO video_events SELECT * FROM video_events;
```


```sql
OPTIMIZE TABLE video_events_agg FINAL;

SELECT count()
FROM video_events

┌───count()─┐
│ 341072384 │
└───────────┘

SELECT count()
FROM video_events_agg

┌──count()─┐
│ 30841175 │
└──────────┘
```

^^^ Видим, что агрегация дала нам некоторый профит.

Запросы выполняются за приемлемое время

```sql

SELECT count()
FROM video_events_agg
WHERE user_id = 7

┌─count()─┐
│  629508 │
└─────────┘
1 rows in set. Elapsed: 0.011 sec. Processed 1.12 million rows, 4.48 MB (103.74 million rows/s., 414.95 MB/s.)

SELECT count()
FROM video_events_agg
WHERE video_id = 7000

┌─count()─┐
│     633 │
└─────────┘

1 rows in set. Elapsed: 0.033 sec. Processed 12.44 million rows, 99.54 MB (371.37 million rows/s., 2.97 GB/s.)
```

Но, для запросов по конкретному видео у нас читается значительно больше строк чем при запросе по конкретному пользователю.

Рассказать про cost-based оптимизации

Учитывая что мы уверены в том, что конкретное видео всегда соответствует конкретному пользователю (данные связаны) мы можем оптимизировать запрос добавив в него user_id

```sql
SET send_logs_level = 'trace';

SELECT count()
FROM video_events_agg
WHERE (video_id = 7000) AND (user_id = 7)

┌─count()─┐
│     633 │
└─────────┘

1 rows in set. Elapsed: 0.023 sec. Processed 506.20 thousand rows, 3.13 MB (22.27 million rows/s., 137.55 MB/s.)
```

Однако, лучше чтоб подобными вещами занималась СУБД :)


## Экспериментальные возможности

Теперь ClickHouse поддерживает создание дополнительных индексов.

В классическом понимании индексы помогают что-то найти, в ClickHouse же индексы позволяют пропускать блоки и не читать их.

```sql

SET allow_experimental_data_skipping_indices = 1; -- возможность пока экспериментальная, надо включать
ALTER TABLE video_events_agg ADD INDEX idx_video_id video_id TYPE minmax GRANULARITY 3;
OPTIMIZE TABLE video_events_agg FINAL;

-- Проверим
SELECT count()
FROM video_events_agg
WHERE video_id = 7000

┌─count()─┐
│     633 │
└─────────┘

1 rows in set. Elapsed: 0.009 sec. Processed 663.55 thousand rows, 5.31 MB (75.90 million rows/s., 607.20 MB/s.)

[kshvakov] 2019.05.07 08:10:51.718477 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Debug> executeQuery: (from 127.0.0.1:53160) SELECT count() FROM video_events_agg WHERE video_id = 7000
[kshvakov] 2019.05.07 08:10:51.720669 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Debug> default.video_events_agg (SelectExecutor): Key condition: (column 2 in [7000, 7000])
[kshvakov] 2019.05.07 08:10:51.720752 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Debug> default.video_events_agg (SelectExecutor): MinMax index condition: unknown
[kshvakov] 2019.05.07 08:10:51.735222 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Debug> default.video_events_agg (SelectExecutor): Index `idx_video_id` has dropped 1039 granules.
[kshvakov] 2019.05.07 08:10:51.736722 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Debug> default.video_events_agg (SelectExecutor): Index `idx_video_id` has dropped 209 granules.
[kshvakov] 2019.05.07 08:10:51.736824 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Debug> default.video_events_agg (SelectExecutor): Selected 2 parts by date, 2 parts by key, 249 marks to read from 236 ranges
[kshvakov] 2019.05.07 08:10:51.737111 {4fe3e3c0-a911-4f05-811b-350efd806528} [ 48 ] <Trace> default.video_events_agg (SelectExecutor): Reading approx. 2039808 rows with 4 streams
```

Логи, они должны быть отсортированы в обратном хронологическом порядке (мы должны видеть последние действия)

```sql
SELECT *
FROM video_events
WHERE (video_id = 7000) AND (user_id = 7)
ORDER BY event_time DESC
LIMIT 10

┌──────────event_time─┬─user_id─┬─video_id─┬──────bytes─┬─os──────┬─device──┬─browser─┬─country─┬─domain────────────┐
│ 2019-05-05 19:07:23 │       7 │     7000 │ 2947257000 │ OS X    │ Tablet  │ Chrome  │ UA      │ exness.com        │
│ 2019-05-05 19:07:23 │       7 │     7000 │ 2947257000 │ OS X    │ Tablet  │ Chrome  │ UA      │ exness.com        │
│ 2019-05-05 19:05:02 │       7 │     7000 │  703196000 │ Linux   │ Mobile  │ IE      │ BY      │ altinity.com      │
│ 2019-05-05 19:05:02 │       7 │     7000 │  703196000 │ Linux   │ Mobile  │ IE      │ BY      │ altinity.com      │
│ 2019-05-05 18:45:51 │       7 │     7000 │ 1660805000 │ Linux   │ Mobile  │ IE      │ RU      │ altinity.com      │
│ 2019-05-05 18:45:51 │       7 │     7000 │ 1660805000 │ Linux   │ Mobile  │ IE      │ RU      │ altinity.com      │
│ 2019-05-05 18:42:43 │       7 │     7000 │ 2015503000 │ Windows │ Desktop │ Firefox │ CY      │ clickhouse.yandex │
│ 2019-05-05 18:42:43 │       7 │     7000 │ 2015503000 │ Windows │ Desktop │ Firefox │ CY      │ clickhouse.yandex │
│ 2019-05-05 18:38:30 │       7 │     7000 │  950649000 │ OS X    │ Tablet  │ Chrome  │ CY      │ exness.com        │
│ 2019-05-05 18:38:30 │       7 │     7000 │  950649000 │ OS X    │ Tablet  │ Chrome  │ CY      │ exness.com        │
└─────────────────────┴─────────┴──────────┴────────────┴─────────┴─────────┴─────────┴─────────┴───────────────────┘

10 rows in set. Elapsed: 3.327 sec. Processed 341.07 million rows, 1.54 GB (102.52 million rows/s., 463.40 MB/s.)
```

Учитывая то, что мы уже накопили статистических данных в таблице с агрегатами грех этим не воспользоваться

```sql
SELECT toDateTime(date) + INTERVAL hour HOUR AS from, max_time AS to, runningAccumulate(state) AS count FROM (
    SELECT date, hour, max_time, state FROM (
        SELECT
            date
            , timeMap.hour            AS hour
            , sumState(timeMap.count) AS state
        FROM video_events_agg ARRAY JOIN timeMap
        WHERE video_id = 7000 AND user_id = 7
        GROUP BY date, hour
        ORDER BY date DESC, hour DESC
    ) ANY JOIN (
        SELECT
            date
            , maxMerge(max_time) AS max_time
        FROM video_events_agg
        WHERE video_id = 7000 AND user_id = 7
        GROUP BY date
    ) USING (date)
) WHERE count > 10 LIMIT 1

┌────────────────from─┬──────────────────to─┬─count─┐
│ 2019-05-05 18:00:00 │ 2019-05-05 19:07:23 │    12 │
└─────────────────────┴─────────────────────┴───────┘

1 rows in set. Elapsed: 0.051 sec. Processed 983.04 thousand rows, 13.33 MB (19.41 million rows/s., 263.27 MB/s.)
```

Мы получили интервал времени в котором есть необходимые нам события, пробуем оптимизировать запрос

```sql

SELECT *
FROM video_events
WHERE (video_id = 7000) AND (user_id = 7)
AND event_time >= '2019-05-05 18:00:00' AND event_time <= '2019-05-05 19:07:23'
ORDER BY event_time DESC
LIMIT 10

┌──────────event_time─┬─user_id─┬─video_id─┬──────bytes─┬─os──────┬─device──┬─browser─┬─country─┬─domain────────────┐
│ 2019-05-05 19:07:23 │       7 │     7000 │ 2947257000 │ OS X    │ Tablet  │ Chrome  │ UA      │ exness.com        │
│ 2019-05-05 19:07:23 │       7 │     7000 │ 2947257000 │ OS X    │ Tablet  │ Chrome  │ UA      │ exness.com        │
│ 2019-05-05 19:05:02 │       7 │     7000 │  703196000 │ Linux   │ Mobile  │ IE      │ BY      │ altinity.com      │
│ 2019-05-05 19:05:02 │       7 │     7000 │  703196000 │ Linux   │ Mobile  │ IE      │ BY      │ altinity.com      │
│ 2019-05-05 18:45:51 │       7 │     7000 │ 1660805000 │ Linux   │ Mobile  │ IE      │ RU      │ altinity.com      │
│ 2019-05-05 18:45:51 │       7 │     7000 │ 1660805000 │ Linux   │ Mobile  │ IE      │ RU      │ altinity.com      │
│ 2019-05-05 18:42:43 │       7 │     7000 │ 2015503000 │ Windows │ Desktop │ Firefox │ CY      │ clickhouse.yandex │
│ 2019-05-05 18:42:43 │       7 │     7000 │ 2015503000 │ Windows │ Desktop │ Firefox │ CY      │ clickhouse.yandex │
│ 2019-05-05 18:38:30 │       7 │     7000 │  950649000 │ OS X    │ Tablet  │ Chrome  │ CY      │ exness.com        │
│ 2019-05-05 18:38:30 │       7 │     7000 │  950649000 │ OS X    │ Tablet  │ Chrome  │ CY      │ exness.com        │
└─────────────────────┴─────────┴──────────┴────────────┴─────────┴─────────┴─────────┴─────────┴───────────────────┘

10 rows in set. Elapsed: 0.039 sec. Processed 409.60 thousand rows, 1.85 MB (10.55 million rows/s., 47.72 MB/s.)

```

Итого: 0.051 sec для определения "стоимости" запроса и 0.039 sec на выполнение. 0.09 vs 3.327 sec

# В качестве вывода

Оптимизация основанная на статистике о распределении данных является достаточно мощным инструментом позволяющим уменьшать время отклика системы и снижающим нагрузку.