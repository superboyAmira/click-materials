---
tags:
  - db
  - clickhouse
links:
  - https://blog.duyet.net/2024/06/clickhouse-replicatedreplacingmergetree.html
  - https://clickhouse.com/docs/en/operations/settings/merge-tree-settings#replicated-deduplication-window
  - https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree
---
## Описание движка

Самая главная фича движка - автоматический асинхронный upsert данных и дедубликация [[Brain/DB/ClickHouse/Scheduler|Scheduler]] в фоновом процессе клика [[Merge]] при слиянии partов. Он наследует все свойства MergeTree но с доп логикой -  во время мерджа [[Merge]], если в одной части встречаются несколько строк с одинаковым значением сортировочного ключа, остаётся только одна из них ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=%D0%94%D0%B0%D0%BD%D0%BD%D1%8B%D0%B9%20%D0%B4%D0%B2%D0%B8%D0%B6%D0%BE%D0%BA%20%D0%BE%D1%82%D0%BB%D0%B8%D1%87%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%BE%D1%82%20MergeTree,PRIMARY%20KEY)). Таким образом, постепенно в таблице устраняются дубли - Движок хорош для UPSERT

Особенности:
- **Идентификация дублей:** Критерием уникальности служит выражение `ORDER BY` (то есть набор колонок сортировки) ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=%D0%94%D0%B0%D0%BD%D0%BD%D1%8B%D0%B9%20%D0%B4%D0%B2%D0%B8%D0%B6%D0%BE%D0%BA%20%D0%BE%D1%82%D0%BB%D0%B8%D1%87%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%BE%D1%82%20MergeTree,PRIMARY%20KEY)) ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=%D0%9F%D0%B0%D1%80%D0%B0%D0%BC%D0%B5%D1%82%D1%80%D1%8B%20ReplacingMergeTree)). Если в сливаемых частях есть несколько строк, у которых все значения ключевых столбцов совпадают, то в результирующую часть попадет только одна из них.
- **Выбор записи при дедупликации:** По умолчанию остается **последняя вставленная запись** (из самой свежей части) ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=%D0%9F%D1%80%D0%B8%20%D1%81%D0%BB%D0%B8%D1%8F%D0%BD%D0%B8%D0%B8%20,%D1%81%D0%BE%D1%80%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%BE%D1%87%D0%BD%D1%8B%D0%BC%20%D0%BA%D0%BB%D1%8E%D1%87%D0%BE%D0%BC%20%D0%BE%D1%81%D1%82%D0%B0%D0%B2%D0%BB%D1%8F%D0%B5%D1%82%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D0%BE%D0%B4%D0%BD%D1%83)). Однако ReplacingMergeTree может быть параметризован _номером версии_. Если при создании таблицы указать столбец `version` (типа UInt/DateTime) в скобках, то при слиянии будет сохранена строка с максимальным значением версии ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=,%D0%BE%D1%81%D1%82%D0%B0%D0%BD%D0%B5%D1%82%D1%81%D1%8F%20%D1%81%D0%B0%D0%BC%D0%B0%D1%8F%20%D0%BF%D0%BE%D1%81%D0%BB%D0%B5%D0%B4%D0%BD%D1%8F%D1%8F%20%D0%B2%D1%81%D1%82%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%BD%D0%B0%D1%8F%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B0)). (например, метку времени как версию).
- `ver` — столбец с номером версии. Тип `UInt*`, `Date`, `DateTime` или `DateTime64`. Необязательный параметр.
> При слиянии `ReplacingMergeTree` оставляет только строку для каждого уникального ключа сортировки:
	 _Последнюю в выборке, если `ver` не задан._ Под выборкой здесь понимается набор строк в наборе кусков данных, участвующих в слиянии. Последний по времени создания кусок (последняя вставка) будет последним в выборке. Таким образом, после дедупликации для каждого значения ключа сортировки останется самая последняя строка из самой последней вставки.
	_С максимальной версией, если `ver` задан._ Если `ver` одинаковый у нескольких строк, то для них используется правило -- если `ver` не задан, т.е. в результате слияния останется самая последняя строка из самой последней вставки.
- **Поведение во времени:** Важно понимать, что дедупликация происходит _только в момент мерджа_. Пока дублирующие строки находятся в разных частях (до их слияния), они обе видны в запросах. Поэтому ReplacingMergeTree не гарантирует отсутствие дублей _немедленно_ после вставки – система уберет их eventually (в конечном итоге). Принудительно запустить слияние можно командой `OPTIMIZE TABLE ... FINAL`, но это тяжеловесная операция для больших таблиц, не рассчитанная на постоянное использование ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=%D0%94%D0%B5%D0%B4%D1%83%D0%BF%D0%BB%D0%B8%D0%BA%D0%B0%D1%86%D0%B8%D1%8F%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85%20%D0%BF%D1%80%D0%BE%D0%B8%D1%81%D1%85%D0%BE%D0%B4%D0%B8%D1%82%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D0%B2%D0%BE,%D0%B8%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D1%82%D1%8C%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%BE%D0%B5%20%D0%BA%D0%BE%D0%BB%D0%B8%D1%87%D0%B5%D1%81%D1%82%D0%B2%D0%BE%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85)).
- **Удаление устаревших данных:** Движок не предназначен для строгого хранения единственной версии записи. Он лишь экономит место, удаляя дубликаты в фоне. В запросах, требующих гарантированно уникальные результаты, рекомендуется использовать оператор `FINAL` (см. раздел оптимизации) для явного схлопывания дублей на лету ([ReplacingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replacingmergetree#:~:text=%D0%92%D0%BE%20%D0%B2%D1%80%D0%B5%D0%BC%D1%8F%20%D1%81%D0%BB%D0%B8%D1%8F%D0%BD%D0%B8%D1%8F%20ReplacingMergeTree%20%D0%B2%D1%8B%D1%8F%D0%B2%D0%BB%D1%8F%D0%B5%D1%82,%D1%83%D0%B4%D0%B0%D0%BB%D0%B5%D0%BD%D0%B8%D1%8F%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%2C%20%D1%83%D1%87%D0%B8%D1%82%D1%8B%D0%B2%D0%B0%D0%B5%D0%BC%D1%8B%D1%85%20%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%B0%D1%85)).

**Пример использования:** Предположим, есть поток событий обновления профиля пользователя. Каждый раз при изменении профиля приходит новая запись, полностью замещающая предыдущую. Можно хранить такие события в ReplacingMergeTree по ключу `UserID`, указав `version` – время обновления. Тогда в результате слияний для каждого пользователя в таблице останется только одна строка – с последней версией профиля. Устаревшие версии будут автоматически отсеяны^

Создание такой таблицы:

```sql
CREATE TABLE user_profiles (
    UserID   UInt64,
    Name     String,
    Age      UInt8,
    Updated  DateTime
)
ENGINE = ReplacingMergeTree(Updated)   -- используем колонку в качестве версии
ORDER BY UserID;
```

При слияниях для каждого `UserID` останется запись с максимальным `Updated`.

---
## Миграция из Postgres подобных

при миграции из транзакционной базы данных, такой как Postgres, исходный первичный ключ Postgres должен быть включен в предложение Clickhouse ORDER BY.

---
## FINAL предикат

Если в запросе используется модификатор `FINAL`, то ClickHouse полностью мёржит данные перед выдачей результата, таким образом выполняя все преобразования данных, которые производятся движком таблиц при мёржах.

Он применим при выборе данных из таблиц, использующих [MergeTree](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/mergetree)- семейство движков. Также поддерживается для:

- [Replicated](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replication) варианты исполнения `MergeTree` движков.
- [View](https://clickhouse.com/docs/ru/engines/table-engines/special/view), [Buffer](https://clickhouse.com/docs/ru/engines/table-engines/special/buffer), [Distributed](https://clickhouse.com/docs/ru/engines/table-engines/special/distributed), и [MaterializedView](https://clickhouse.com/docs/ru/engines/table-engines/special/materializedview), которые работают поверх других движков, если они созданы для таблиц с движками семейства `MergeTree`.

Теперь `SELECT` запросы с `FINAL` выполняются параллельно и, следовательно, немного быстрее. Но имеются серьезные недостатки при их использовании (смотрите ниже). Настройка [max_final_threads](https://clickhouse.com/docs/ru/operations/settings/settings#max-final-threads) устанавливает максимальное количество потоков.

_do_not_merge_across_partitions_select_final=1_ позволяет партиционированной таблице выполнять принудительное слияние FINAL паралелльно в таблице
### Недостатки[​](https://clickhouse.com/docs/ru/sql-reference/statements/select/from#drawbacks "Direct link to Недостатки")

Запросы, которые используют `FINAL` выполняются немного медленее, чем аналогичные запросы без него, потому что:

- Данные мёржатся во время выполнения запроса в памяти, и это не приводит к физическому мёржу кусков на дисках.
- Запросы с модификатором `FINAL` читают столбцы первичного ключа в дополнение к столбцам, используемым в запросе.

**В большинстве случаев избегайте использования `FINAL`.** Общий подход заключается в использовании агрегирующих запросов, которые предполагают, что фоновые процессы движков семейства `MergeTree` ещё не случились (например, сами отбрасывают дубликаты).

### replicated_deduplication_window[​](https://clickhouse.com/docs/en/operations/settings/merge-tree-settings#replicated-deduplication-window "Direct link to replicated_deduplication_window")

The number of most recently inserted blocks for which ClickHouse Keeper stores hash sums to check for duplicates.

Possible values:

- Any positive integer.
- 0 (disable deduplication)

Default value: 1000.

The `Insert` command creates one or more blocks (parts). For [insert deduplication](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication), when writing into replicated tables, ClickHouse writes the hash sums of the created parts into ClickHouse Keeper. Hash sums are stored only for the most recent `replicated_deduplication_window` blocks. The oldest hash sums are removed from ClickHouse Keeper. A large number of `replicated_deduplication_window` slows down `Inserts` because it needs to compare more entries. The hash sum is calculated from the composition of the field names and types and the data of the inserted part (stream of bytes).
