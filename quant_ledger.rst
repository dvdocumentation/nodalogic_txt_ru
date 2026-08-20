Quant Ledger
============

``quant_ledger`` — дополнительный серверный SQL-модуль NodaLogic для хранения количественных остатков и движений. Он подходит для складских остатков, денег, резервов, загрузки ячеек и других задач, где значения изменяются движениями ``+/-``, а текущее состояние должно вычисляться надежно и атомарно.

Модуль не имеет отдельного интерфейса в Designer и используется только из серверных Python-обработчиков. Если каталог ``quant_ledger`` присутствует в проекте, необходимые SQL-таблицы создаются при запуске приложения. На текущий момент модуль работает с SQLite.

Подключение
-----------

Обычно достаточно импортировать только используемые функции:

.. code-block:: Python

 from quant_ledger.api import (
     quant, move, transaction,
     balance, balances, movements, statement,
     NegativeBalanceError,
 )

Основные понятия
----------------

**space** — логическое пространство учета. Например, детальные складские остатки можно хранить в ``"stock.detail"``, а загрузку ячеек — в отдельном ``"location.load"``.

**quant** — полный аналитический ключ остатка. Именно сочетание ``space + quant`` определяет одну строку остатка.

Например, для склада полный ключ можно построить из склада, товара, партии и ячейки:

.. code-block:: Python

 full_quant = quant(warehouse_id, product_id, lot_id, location_id)

**selector_quant** — дополнительный, более грубый ключ для быстрого отбора группы остатков. Например, все партии и ячейки одного товара на одном складе:

.. code-block:: Python

 selector = quant(warehouse_id, product_id)

``selector_quant`` не является частью уникального ключа остатка, а используется как отдельный индекс для выборок. Для уже существующего полного ``quant`` его менять нельзя.

**resources** — список числовых ресурсов движения. Поддерживается до 16 ресурсов. В складском примере ресурс ``0`` обычно означает количество, ресурс ``1`` — сумму:

.. code-block:: Python

 [quantity, amount]

Значения возвращаются как ``Decimal``. Внутри SQL они хранятся как fixed-point с точностью 6 знаков после запятой.

Простое движение
----------------

Функция ``move()`` одновременно записывает движение и изменяет остаток. Операция атомарна: движение и остаток либо сохраняются вместе, либо не сохраняются вовсе.

Пример прихода:

.. code-block:: Python

 result = move(
     "stock.detail",
     quant(warehouse_id, product_id, lot_id, location_id),
     document_date,
     f"{document_id}:receipt",
     {
         "document": document_id,
         "employee": employee_id,
         "comment": "Приход товара",
     },
     [quantity, amount],
     selector_quant=quant(warehouse_id, product_id),
     allow_negative=False,
 )

Параметры:

 * **space** — пространство учета;
 * **quant** — полный аналитический ключ;
 * **period** — дата/время движения;
 * **operation_id** — стабильный идентификатор логического движения;
 * **details** — произвольные данные движения для истории и фильтрации;
 * **resources** — дельты ресурсов;
 * **selector_quant** — ключ быстрого группового отбора;
 * **allow_negative** — разрешать ли отрицательный ресурс ``0``.

Для расхода достаточно передать отрицательную дельту:

.. code-block:: Python

 move(
     "stock.detail",
     full_quant,
     document_date,
     f"{document_id}:expense",
     {"document": document_id},
     [-quantity, -amount],
     selector_quant=selector,
     allow_negative=False,
 )

По умолчанию отрицательный ресурс ``0`` запрещен. При нехватке остатка возникает ``NegativeBalanceError``:

.. code-block:: Python

 try:
     move(...)
 except NegativeBalanceError as exc:
     Message("Недостаточно остатка: " + str(exc.attempted))

Если нужно запретить отрицательные значения сразу для нескольких ресурсов, можно указать их индексы явно:

.. code-block:: Python

 move(..., nonnegative_resources=[0, 1])

Повторное проведение
--------------------

``operation_id`` должен быть стабильным для одного логического движения. Если ``move()`` вызывается повторно с тем же ``operation_id``, старое движение не дублируется и не игнорируется — оно заменяется новым в одной транзакции.

.. code-block:: Python

 result = move(
     "stock.detail",
     full_quant,
     document_date,
     f"{document_id}:receipt",
     {"document": document_id},
     [new_quantity, new_amount],
     selector_quant=selector,
 )

 if result.reposted:
     Message("Документ перепроведен")

При перепроведении могут измениться период, полный ``quant``, ``selector_quant``, ``details`` и ресурсы. Если новый вариант приводит к ошибке, например к отрицательному остатку, операция откатывается и прежнее движение остается без изменений.

Несколько движений одной операцией
----------------------------------

Если бизнес-операция состоит из нескольких движений, их следует объединять через ``transaction()``. Типичный пример — перемещение между ячейками: расход с одной ячейки и приход на другую должны пройти только вместе.

.. code-block:: Python

 source_quant = quant(warehouse_id, product_id, lot_id, source_location_id)
 target_quant = quant(warehouse_id, product_id, lot_id, target_location_id)
 selector = quant(warehouse_id, product_id)

 with transaction() as tx:
     tx.move(
         "stock.detail",
         source_quant,
         document_date,
         f"{document_id}:source",
         {"document": document_id},
         [-quantity],
         selector_quant=selector,
         allow_negative=False,
     )

     tx.move(
         "stock.detail",
         target_quant,
         document_date,
         f"{document_id}:target",
         {"document": document_id},
         [quantity],
         selector_quant=selector,
         allow_negative=False,
     )

Если первое или второе движение не может быть выполнено, откатывается вся транзакция. Для разных движений одной операции используются разные ``operation_id``, например ``:source`` и ``:target``.

Чтение текущих остатков
-----------------------

**balance(space, quant)** — получить точный остаток по полному ключу:

.. code-block:: Python

 row = balance("stock.detail", full_quant)
 qty = row.resources[0]
 amount = row.resources[1]

Если остатка еще нет, возвращается строка с нулевыми ресурсами.

**balances(...)** — выбрать несколько остатков. Наиболее эффективный вариант — отбор по ``selector_quant``:

.. code-block:: Python

 rows = balances(
     "stock.detail",
     selector_quant=quant(warehouse_id, product_id),
     positive_resource=0,
 )

 for row in rows:
     warehouse, product, lot, location = row.parts
     qty = row.resources[0]

Полезные параметры ``balances()``:

 * **selector_quant** — быстрый отбор по селектору;
 * **quants** — набор точных полных квантов;
 * **nonzero_resource** — оставить строки, где указанный ресурс не равен нулю;
 * **positive_resource** — оставить строки с положительным ресурсом;
 * **limit** — ограничить количество строк.

История движений
----------------

**movements()** возвращает историю движений и позволяет фильтровать ее по периоду, ``quant``, ``selector_quant``, ``operation_id`` и полям ``details``.

.. code-block:: Python

 rows = movements(
     "stock.detail",
     period_from=date_from,
     period_to=date_to,
     details={"document": document_id},
 )

 for row in rows:
     print(row.period, row.resources, row.details)

``details`` предназначен для аудита и точной фильтрации движений. Например, в нем удобно хранить UID документа, сотрудника, тип операции или комментарий.

Ведомость за период
-------------------

Для отчетов вида «Начальный остаток / Приход / Расход / Конечный остаток» используется ``statement()``. Нет необходимости загружать все движения и пересчитывать итоги вручную.

.. code-block:: Python

 rows = statement(
     "stock.detail",
     period_from=date_from,
     period_to=date_to,
     selector_quant=quant(warehouse_id, product_id),
 )

 for row in rows:
     opening = row.opening[0]
     income = row.income[0]
     expense = row.expense[0]
     closing = row.closing[0]

Каждая строка ``StatementRow`` относится к одному полному ``quant``. Если отчету нужен более высокий уровень, например только склад + товар, строки следует агрегировать в Python после SQL-выборки.

FEFO и другие бизнес-сортировки
-------------------------------

В ``quant`` находятся только ключи, а не поля связанных узлов. Поэтому SQL не может, например, сортировать партии по ``Lot.expire_date``, если в кванте хранится только UID партии.

Правильный подход для FEFO:

.. code-block:: Python

 rows = balances(
     "stock.detail",
     selector_quant=quant(warehouse_id, product_id),
     positive_resource=0,
 )

 lot_ids = {str(row.parts[2]) for row in rows}
 lots = Lot.get_many_data(lot_ids)

 rows.sort(
     key=lambda row: str(
         (lots.get(str(row.parts[2])) or {}).get("expire_date") or "9999-12-31"
     )
 )

Сначала SQL быстро отбирает кандидатов по ``selector_quant``, затем связанные узлы загружаются пакетно и бизнес-сортировка выполняется в Python.

Работа с quant
--------------

``quant()`` создает канонический обратимый ключ. Разбирать его вручную по символу ``|`` не нужно.

.. code-block:: Python

 q = quant(warehouse_id, product_id, lot_id, location_id)
 parts = parse_quant(q)
 lot_id = quant_part(q, 2)

Для этого при необходимости импортируются ``parse_quant`` и ``quant_part``.

Если в вашей модели используется специальное значение ``"~"`` для пустого измерения, оно является обычным точным значением, а не wildcard:

.. code-block:: Python

 q = quant(warehouse_id, product_id, "~", location_id)

Проверка и восстановление
-------------------------

**verify_space(space)** сравнивает сохраненные остатки с суммой движений:

.. code-block:: Python

 from quant_ledger.api import verify_space

 result = verify_space("stock.detail")
 if not result.valid:
     print(result.errors)

**rebuild_balances(space)** полностью перестраивает остатки пространства по истории движений и затем выполняет проверку:

.. code-block:: Python

 from quant_ledger.api import rebuild_balances

 result = rebuild_balances("stock.detail")

Эти функции предназначены для диагностики и восстановления, а не для обычного проведения документов.

Scope конфигурации
------------------

В серверном обработчике NodaLogic ``scope`` текущей конфигурации определяется автоматически, поэтому обычно передавать его не требуется.

При использовании модуля вне серверного обработчика ``scope`` нужно задать явно:

.. code-block:: Python

 row = balance("stock.detail", full_quant, scope=config_uid)

Данные разных конфигураций с одинаковыми ``space`` и ``quant`` остаются изолированными друг от друга.

Практические правила
--------------------

 * Не изменяйте SQL-таблицы остатков напрямую — проводите только дельты через ``move()``.
 * Не вычисляйте новый остаток в приложении по схеме «прочитать + изменить + сохранить».
 * Используйте стабильный ``operation_id`` для перепроведения документа.
 * Используйте ``transaction()`` для операций, которые должны проводиться целиком.
 * Для быстрых групповых выборок заранее продумайте ``selector_quant``.
 * Не меняйте ``selector_quant`` для уже существующего полного ``quant``.
 * Для текущего остатка используйте ``balance()``/``balances()``, для истории — ``movements()``, для периодической ведомости — ``statement()``.
 * Не вызывайте ``quant_ledger`` из Android-обработчиков — модуль предназначен только для сервера.
