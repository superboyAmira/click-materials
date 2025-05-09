---
tags:
  - db
  - clickhouse
---

**AggregatingMergeTree** идёт дальше SummingMergeTree. Он позволяет сохранять _промежуточные состояния агрегатных функций_ и объединять их при слиянии. Обычно используется вместе со специальными функциями вида `argMaxState`, `sumState`, `uniqState` и т.п., а затем при чтении применяется `argMaxMerge`, `sumMerge`, `uniqExactMerge` и пр.  Короче - имеет смысл, если агрегация сокращает объем данных на порядок и более ([AggregatingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/aggregatingmergetree#:~:text=match%20at%20L240%20%D0%98%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20,%D1%83%D0%BC%D0%B5%D0%BD%D1%8C%D1%88%D0%B0%D0%B5%D1%82%20%D0%BA%D0%BE%D0%BB%D0%B8%D1%87%D0%B5%D1%81%D1%82%D0%B2%D0%BE%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%20%D0%BD%D0%B0%20%D0%BF%D0%BE%D1%80%D1%8F%D0%B4%D0%BA%D0%B8)). Он снижает нагрузку на чтение (меньше строк), перекладывая часть вычислений на время записи.
Практически, это движок для реализации _инкрементной агрегации_: когда мы на этапе вставки сразу вычисляем некие агрегаты и сохраняем их а при чтении лишь домердживаем

Особенности AggregatingMergeTree:

- **Хранение агрегатных состояний:** Должны использоваться типы колонок _AggregateFunction_ или специальные агрегатные типы. Например, можно определить колонку `hits AggregateFunction(sum, UInt64)` – и при вставке в нее записывать результат `sumState(value)` для группы значений. AggregatingMergeTree будет хранить эти состояния 
- **Объединение при слиянии:** При мерджах движок объединяет состояния агрегатных функций – фактически вызывая соответствующие [[Merge]] функции. Например, если в двух частях есть _SUM_ для одной группы, то при слиянии получится одно состояние, эквивалентное сумме обоих. Таким образом, данные агрегируются инкрементно по мере слияния частей.
- **Запросы к AggregatingMergeTree:** Чтобы получить финальные значения, необходимо в запросе использовать агрегатные функции с суффиксом `-Merge` и `GROUP BY` по ключу агрегирования ([AggregatingMergeTree | ClickHouse Docs](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/aggregatingmergetree#:~:text=%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%D0%BC%D0%B8%20,Merge)). То есть по сути сделать финальный шаг агрегации. Без этого запроса будут выданы внутренние данные, без доп мержа

Таким образом, мы храним агрегированные данные, которые при необходимости можно доагрегировать.
