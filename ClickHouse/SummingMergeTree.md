---
tags:
  - db
  - clickhouse
---

**SummingMergeTree** автоматически _агрегирует (суммирует) числовые поля_ при объединении частей. Это полезно, когда в таблицу поступают несводные данные, которые можно частично агрегировать для экономии места.

Как это работает: при слиянии [[Merge]] вместо просто копирования строк он **складывает значения определенных столбцов для строк с одинаковым ключом сортировки и заменяет их одной строкой** ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=%D0%94%D0%B2%D0%B8%D0%B6%D0%BE%D0%BA%20%D0%BD%D0%B0%D1%81%D0%BB%D0%B5%D0%B4%D1%83%D0%B5%D1%82%D1%81%D1%8F%20%D0%BE%D1%82%20MergeTree,%D0%BA%D0%BE%D0%BB%D0%BE%D0%BD%D0%BE%D0%BA%20%D1%81%20%D1%87%D0%B8%D1%81%D0%BB%D0%BE%D0%B2%D1%8B%D0%BC%20%D1%82%D0%B8%D0%BF%D0%BE%D0%BC%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85)) 
если в части встретились несколько строк, имеющие один и тот же ключ (например, один и тот же `CounterID` и `Date` с презентации), то они объединяются: числовые поля суммируются, а вместо группы строк остается одна свернутая строка.

Особенности:
- **Выбор столбцов для суммирования:** При создании таблицы можно опционально указать параметр `columns` – кортеж имен столбцов, которые нужно суммировать. Эти столбцы должны быть числовых типов и не входить в первичный ключ ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=%60columns%60%20,%D0%B4%D0%BE%D0%BB%D0%B6%D0%BD%D1%8B%20%D0%B2%D1%85%D0%BE%D0%B4%D0%B8%D1%82%D1%8C%20%D0%B2%20%D0%BF%D0%B5%D1%80%D0%B2%D0%B8%D1%87%D0%BD%D1%8B%D0%B9%20%D0%BA%D0%BB%D1%8E%D1%87)). Если параметр `columns` не указан, ClickHouse по умолчанию суммирует _все_ числовые колонки, не входящие в ключ ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=%D0%95%D1%81%D0%BB%D0%B8%20,%D0%BD%D0%B5%20%D0%B2%D1%85%D0%BE%D0%B4%D1%8F%D1%82%20%D0%B2%20%D0%BF%D0%B5%D1%80%D0%B2%D0%B8%D1%87%D0%BD%D1%8B%D0%B9%20%D0%BA%D0%BB%D1%8E%D1%87)).
- **Объединение записей:** На этапе мерджа [[Merge]] для каждого уникального ключа сортировки (гранулы) SummingMergeTree складывает значения выбранных столбцов во всех строках этой гранулы и оставляет одну строку ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=match%20at%20L282%20%D0%BF%D0%B5%D1%80%D0%B2%D0%B8%D1%87%D0%BD%D1%8B%D0%BC%20%D0%BA%D0%BB%D1%8E%D1%87%D0%BE%D0%BC,%D0%B4%D0%BB%D1%8F%20%D0%BA%D0%B0%D0%B6%D0%B4%D0%BE%D0%B9%20%D1%80%D0%B5%D0%B7%D1%83%D0%BB%D1%8C%D1%82%D0%B8%D1%80%D1%83%D1%8E%D1%89%D0%B5%D0%B9%20%D1%87%D0%B0%D1%81%D1%82%D0%B8%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85)). Нечисловые столбцы (которые не суммируются и не входят в ключ) заполняются произвольным из значений  ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=%D0%95%D1%81%D0%BB%D0%B8%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F%20%D0%B2%D0%BE%20%D0%B2%D1%81%D0%B5%D1%85%20%D0%BA%D0%BE%D0%BB%D0%BE%D0%BD%D0%BA%D0%B0%D1%85,%D1%81%D1%83%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F%20%D0%B1%D1%8B%D0%BB%D0%B8%200%2C%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B0%20%D1%83%D0%B4%D0%B0%D0%BB%D1%8F%D0%B5%D1%82%D1%81%D1%8F)). Если все суммируемые значения оказались нулевыми – результирующая строка может быть вообще удалена ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=%D0%97%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F%20%D0%B2%20%D0%BA%D0%BE%D0%BB%D0%BE%D0%BD%D0%BA%D0%B0%D1%85%20%D1%81%20%D1%87%D0%B8%D1%81%D0%BB%D0%BE%D0%B2%D1%8B%D0%BC,columns)) (схлопнута подностью).
- **Неполная агрегация:** Важно понимать, что SummingMergeTree агрегирует данные _лишь в пределах частей, которые сливаются_. Если данные с одинаковыми ключами пока находятся в разных частях, они не суммируются до тех пор, пока эти части не объединятся. Поэтому финальная агрегация может быть не полной, пока не произошли все необходимые слияния. Например, запрос селект без явной агрегации может вернуть промежуточные несуммированные значения, если записи еще не слились. Поэтому при чтении данных зачастую **стоит всё равно использовать агрегирующие функции** для получения окончательных сумм ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=ClickHouse%20%D0%BC%D0%BE%D0%B6%D0%B5%D1%82%20%D0%BD%D0%B5%20%D0%BF%D0%BE%D0%BB%D0%BD%D0%BE%D1%81%D1%82%D1%8C%D1%8E%20%D1%81%D1%83%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D1%82%D1%8C,%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%B5)) ([SummingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/summingmergetree#:~:text=match%20at%20L287%20%D1%82,%D0%BA%D0%B0%D0%BA%20%D0%BE%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%BE%20%D0%B2%20%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80%D0%B5%20%D0%B2%D1%8B%D1%88%D0%B5)). SummingMergeTree лишь сокращает объем хранимых данных и снижает нагрузку, частично агрегируя их
- **Типичные применения:** счетчики, метрики, времени – когда приходят по несколько записей для одних и тех же ключей, и их можно сложить
Пример создания SummingMergeTree, суммирующего определенные поля:

```sql
CREATE TABLE metrics_daily (
    Date       Date,
    UserID     UInt64,
    Views      UInt32,
    Clicks     UInt32
)
ENGINE = SummingMergeTree(Views, Clicks)   -- суммируем эти два поля
PARTITION BY toYYYYMM(Date)
ORDER BY (Date, UserID);
```
