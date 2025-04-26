![[Pasted image 20250426034156.png]]
# S3


| Что даёт                                       | Почему это полезно именно в ClickHouse                                                                                                  |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Дешёвое и (почти) бесконечное хранилище**    | SSD‑узлы под «горячие» данные дорогие. S3 позволяет **слить холодные партиции** (старые месяцы/годы) и платить копейки.                 |
| **Отделение compute ↔ storage**                | Нужна дополнительная мощность — поднимаешь новые ноды без переноса данных. Parts лежат в S3, серверы лишь кэшируют востребованное.      |
| **Съём бэкапов «как есть»**                    | S3 уже версионирует объекты; достаточно «снэпшотнуть» метаданные и указать на бакет.                                                    |
| **Zero‑ETL анализ “data‑lake”**                | `SELECT * FROM s3('s3://bucket/logs/*.parquet') …` — можно **подписать** ClickHouse прямо на Parquet/CSV, не копируя на локальный диск. |
| **Архивация с TTL TO VOLUME 's3'**             | В TTL‑политике указываешь `MOVE TO VOLUME 's3' AFTER 90 DAY` — старые части сами уезжают в бакет.                                       |
| **Disaster Recovery / Multi‑Region**           | Данные в объектке легко шарятся между кластерами; потерял ноды → развернул новые, подмонтировал бакет → сразу в строю.                  |
| **ClickHouse Cloud ― “под капотом” именно S3** | Если потом захочешь мигрировать в облачную managed‑версию, структура на S3 уже «родная».                                                |

---

#### Типовые сценарии

1. **Тиринг**  
    _30 дней на SSD MergeTree, дальше — `volume s3`_ → аналитика по свежим данным быстрая, архив доступен (медленнее, но дёшево).
2. **Data‑Lake → ClickHouse**  
    ETL не нужен: `s3`‑table‑function + Materialized View → MergeTree ⇒ получаешь near‑real‑time витрину.
3. **Бэкапы / миграции**  
    Слил `BACKUP TABLE … TO S3`, любой кластер/регион поднимается restore‑ом за минуты.
4. **«Без‑дисковый» read‑only кластер**  
    Веб‑аналитика: храним petabyte click‑stream в S3, подтягиваем фрагменты Parquet, кэшируем лишь рабочие подвыборки.
    


---

>  S3 + ClickHouse = «дёшево храню терабайты, быстро считаю то, что актуально, и не парюсь с бэкапами и масштабированием».


# Hadoop + ClickHouse

> **«Hadoop + ClickHouse»** — это сочетание дешёвого, распределённого хранилища для «сырых» данных и сверхбыстрой column‑store СУБД для OLAP‑аналитики.  
> Импорт/экспорт больших архивов осуществляется через встроенные **HDFS‑движки** и table‑function’ы, что даёт гибкость, параллелизм и минимальные ETL‑затраты.

1. **Data Lake на HDFS**  
    – Все сырые данные (логи, терабайты событий, дампы) лежат в HDFS-последовательностях или как Parquet/ORC-файлы.  
    – HDFS даёт дешёвое распределённое хранилище, надёжность (репликация блоков) и масштабируемость.
    
2. **ClickHouse как витрина**  
    – ClickHouse-файлы (MergeTree‑партиции) хранят «горячие» данные для OLAP‑запросов.  
    – При необходимости ClickHouse берёт фонды из HDFS «на лету» или по расписанию.
    
3. **ETL‑слой**  
    – Spark/Flink/MapReduce читают данные из HDFS, приводят к единой схеме, пишут Parquet/CSV.  
    – Затем ClickHouse «подхватывает» эти файлы через HDFS‑движок или table‑function.
    
4. **Двухсторонняя интеграция**  
    – **Импорт**: HDFS → ClickHouse  
    – **Экспорт** (редко): ClickHouse → HDFS (для downstream‑систем на Hadoop).
    

---

## 2. Импорт больших архивов из HDFS

### 2.1 Через `HDFS` Engine

sql

Копировать

`CREATE TABLE hdfs_logs (     ts        DateTime,     user_id   UInt64,     action    String ) ENGINE = HDFS('hdfs://namenode:9000/path/to/logs/*.parquet', 'Parquet');`

– ClickHouse сразу видит все файлы Parquet в указанном каталоге.  
– Запросы к `hdfs_logs` сканируют файлы на HDFS-посреде.

### 2.2 Через `hdfs` Table Function

sql

Копировать

`SELECT * FROM hdfs(     'hdfs://namenode:9000/path/to/logs/2025-*.orc',     'ORC',     'ts DateTime, user_id UInt64, action String' ) WHERE ts >= today() - 7;`

– Удобно для ad‑hoc‑анализа и временных витрин.  
– Можно комбинировать с `Distributed`‑таблицами для шардинга по узлам.

### 2.3 Параллелизм чтения

– ClickHouse разбивает набор файлов на «split’ы» и параллельно читает их по нескольким потокам.  
– Можно настроить `hdfs_max_connection_pool_size` и прочие параметры в `config.xml`.

---

## 3. Экспорт из ClickHouse в HDFS

### 3.1 `HDFS` Engine + `INSERT`

sql

Копировать

`-- Создаём «холостую» HDFS‑таблицу CREATE TABLE export_users (     user_id   UInt64,     signup_dt DateTime ) ENGINE = HDFS(   'hdfs://namenode:9000/exports/users_${shard}.parquet',    'Parquet' );  -- Записываем туда выборку INSERT INTO export_users SELECT user_id, signup_dt FROM users WHERE signup_dt < today() - 30;`

– ClickHouse создаст Parquet‑файл для каждого шарда (`${shard}`) и зальёт туда строки.

### 3.2 Table Function + `SELECT … INTO OUTFILE`

sql

Копировать

`SELECT * FROM users WHERE signup_dt < today() - 7 INTO OUTFILE 'hdfs://namenode:9000/tmp/users_week.parquet' FORMAT Parquet;`

– Альтернатива: «выгрузить» одним файлом через `INTO OUTFILE`.

---

## 4. Best Practices

1. **Совпадение схемы**  
    – Синхронизируйте структуру колонок между Parquet/ORC и MergeTree‑таблицами.  
    – Используйте одинаковые названия, типы (DateTime64, Decimal и т. д.).
    
2. **Разбиение на файлы**  
    – Создавайте файлы не чересчур маленькими (лучше > 100 МБ) и не гигантскими (> 10 ГБ), чтобы ClickHouse мог эффективно распараллелить чтение.
    
3. **Компрессия**  
    – Parquet/ORC-файлы обычно уже сжаты Snappy/ZSTD. Не стоит дополнительно сжимать на уровне ClickHouse, чтобы избежать дублирования.
    
4. **Параметры HDFS‑движка**  
    – `hdfs_max_connection_pool_size` — число одновременных коннекшенов к NameNode.  
    – `hdfs_read_ahead_buffer_size` — размер предварительной подкачки данных.
    
5. **TTL и Tiered Storage**  
    – Можно настроить `MOVE TO VOLUME 'hdfs_volume' AFTER X DAY` в MergeTree, чтобы старые партиции автоматически перемещались в HDFS.
    

---
