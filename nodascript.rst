.. NodaLogic documentation master file, created by
   sphinx-quickstart on Wed Nov  5 07:29:33 2025.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

NodaScript
=============

NodaScript можно использовать прямо в разметке и также можно использовать в системе событий, указывая его в качестве обработчика событий (как альтернатива python-обработчикам). Это высокопроизводительный нативный интерпретатор собственного языка который работает напрямую и всегда доступен. Его можно использовать в формах без оглядки на производительность. Он работает непосредственно в фронтовой части в Android и в JS коде html-страницы в WEB-клиенте.

Цель использования - сократить кодовую базу, сделать решения более легковесными и читаемыми. На него можно сгрузить все, что касается обработки на интерфейсе, оставив на Python бизнес-логику, интеграции и какие то сложные алгоритмы.
 

Использование в значениях элементов
--------------------------------------

В любом значении визуального элемента, по аналогии с ``@ключ``, можно указать ``#<скрипт return значение>`` Тут может быть любой по сложности скрипт - с циклами, условиями. Главное чтобы был хоть один возврат. 

Пример: ``#return ?(_date.balance<0,'--',_date.balance)``

Пример элемента в 1С-похожей реализации:

 
.. code-block:: JavaScript

  {"type":"Text","value":"#Если _data.qty!=Неопределено и _data.price!=Неопределено Тогда возврат _data.qty*_data.price  Иначе  возврат  '--' КонецЕсли", "gravity":"right"}  


Использование в событиях непосредственно в разметке
------------------------------------------------------

У активных элементов можно прописать обработчик прямо в свойстве элемента в разметке:

``{"type":"Button","id":"btn_update","caption":"Simple button","onClick":"message('btn_update')"}``

Список событий элементов:

 * **Button** : ``onClickParent``(для владельца списка), ``onClick``
 * **Switch** : ``onClickParent``(для владельца списка), ``onClick``
 * **CheckBox** : ``onClickParent``(для владельца списка), ``onClick``
 * **Spinner** : ``onClickParent``(для владельца списка), ``onClick``
 * **Input** : ``onClickParent``(для владельца списка), ``onClick``
 * **NodeInput** : ``onSelect``
 * **DatasetField** : ``onSelect``
 * **Table(событие клика по списку)**  : ``onClick``

Использование в качестве обработчика событий в Событиях
----------------------------------------------------------

Наравне с обработчиками python можно делать обработчики NodaScript в виде подписок на события (в разделе События класса). В этом случае конфигурация получается еще более компактная. Для этого в событии в качестве метода надо выбрать NodaScript и указать код в окне кода.


Описание языка
-------------------

Данные
~~~~~~~~~~~

Все скрипты работают с корневым JSON-объектом _data и меняют его на месте:
 
.. code-block:: JavaScript

  _data.Сумма = 0;
  line = _data.lines[_data.tab1_input_position];
  line.qty = line.qty + 1;


Общее
~~~~~~~

 * Нечувствительность к регистру (Если/ЕСЛИ/если).
 * Разделитель инструкций: `;` (последняя `;` необязательна перед `}` или концом скрипта).
 * Комментарии: // ...
 * Строки: 'одинарные' или "двойные"

Типы значений (строго)
~~~~~~~~~~~~~~~~~~~~~~~~~~~


 * Числа: int и float
 * Строка: string
 * Логический: bool (Истина/Ложь)
 * Дата: date (epochMillis, long)
 * Массив: array
 * Объект: object
 * Пусто / Неопределено (то же, что null)

Логические выражения
~~~~~~~~~~~~~~~~~~~~~~~~

Операторы сравнения:

 `` ==  !=  >  >=  <  <=``

Поддержка: числа, строки, date (по epochMillis). Для array/object сравнение по ссылке (identity).

Логические операторы (только bool):

  И / AND / &&     — логическое И

  ИЛИ / OR / ||    — логическое ИЛИ

  НЕ / NOT / !     — логическое НЕ

Проверка вхождения:

  x В arr          // alias IN

  x IN arr         // EN-форма тоже поддерживается

  "key" В obj      // проверка ключа в объекте

Примеры:

.. code-block:: JavaScript

  
  // сравнение
  _data.qty > 0 И _data.price >= 10;

  // скобки и НЕ
  НЕ (_data.isClosed ИЛИ _data.isDeleted);

  // in (вхождение)
  5 В [1,2,3,5];
  "name" В _data;``

Тернарный оператор
------------------

Обычный:

  x = cond ? a : b;

Как в 1С:

  x = ?(cond, a, b);

Условия (как 1С)
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: JavaScript

  
 Если cond Тогда
     // инструкции;
 ИначеЕсли cond2 Тогда
     // инструкции;
 Иначе
     // инструкции;
 КонецЕсли;

Есть и скобочный вариант:
 
.. code-block:: JavaScript

   if condition { ... } else { ... };


Циклы (как 1С)
~~~~~~~~~~~~~~~~

.. code-block:: JavaScript


 Пока cond Цикл
     // ...
 КонецЦикла;

 Для i = 1 По 10 Цикл   // включительно
     // ...
 КонецЦикла;

 Для Каждого item Из _data.lines Цикл
     // ...
 КонецЦикла;

Также доступны:

.. code-block:: JavaScript
  
  while condition { ... }

  for i = 0; i < 10; i = i + 1 { ... }

  for item in _data.lines { ... }


Прервать / Продолжить
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: JavaScript

  Прервать;

  Продолжить;


Исключения
~~~~~~~~~~~~~~~~~

EN-форма:

.. code-block:: JavaScript
  
  try { ... } except err { ... }
  throw expr;

RU-форма:

.. code-block:: JavaScript

    Попытка { ... } 
  
    Исключение err { ... }

    Вызвать expr;


Возврат (для #-выражений)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~


.. code-block:: JavaScript
  
 Возврат выражение;

 Возврат;  // null

Создание структур/массивов (как 1С)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: JavaScript


 x = Новый Массив;
 s = Новый Структура("Ключ1", Значение1, "Ключ2", Значение2);

А также доступны функции:

.. code-block:: JavaScript

   NewArray()
   NewObject()
   NewStructure(key1, val1, key2, val2, ...)


Литералы JSON (удобно в скриптах)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Объект:

.. code-block:: JavaScript

  o = {a: 1, "b": 2};

Массив:

.. code-block:: JavaScript

  a = [1, 2, 3];

Проверка свойства
~~~~~~~~~~~~~~~~~~~~~
.. code-block:: JavaScript

  HasProperty(obj, key)
  Свойство(obj, key)   // алиас

Методы массива
~~~~~~~~~~~~~~~~~~

.. code-block:: JavaScript

  arr.add(x)
  arr.clear()
  arr.contains(x)

Работа с датами
~~~~~~~~~~~~~~~~~~~

Тип date хранится как epochMillis (long, миллисекунды от 1970-01-01T00:00:00Z).

Создание/парсинг:

.. code-block:: JavaScript

  d = Now();

  // ParseDate: принимает либо epochMillis строкой (только цифры),
  // либо дату/дату-время по шаблону SimpleDateFormat.
  d1 = ParseDate("2026-03-01");
  d2 = ParseDate("01.03.2026", "dd.MM.yyyy");
  d3 = ParseDate("1700000000000");   // epochMillis

Форматирование:

.. code-block:: JavaScript

  s = FormatDate(d, "dd.MM.yyyy HH:mm");

Арифметика:

.. code-block:: JavaScript
  
  d2 = AddDays(d, 5);
  d2 = AddMonths(d, 1);
  d2 = AddYears(d, 1);

Границы периодов:

.. code-block:: JavaScript

  StartOfDay(d)    EndOfDay(d)
  StartOfMonth(d)  EndOfMonth(d)
  StartOfYear(d)   EndOfYear(d)

Сравнение дат:

.. code-block:: JavaScript

  ParseDate("2026-01-01") <= Now();

Примечания по timezone:

Все ParseDate/FormatDate и границы периодов работают в timezone движка (rt.tz / Engine.setTimeZone).

HTTP-обёртка (опционально, синхронная)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

В скрипте:

.. code-block:: JavaScript
   
  r = HTTPRequest("GET", "https://example.com/api/items", {"q":"abc"}, Пусто);
  // с заголовками:
  r = HTTPRequest("GET", "https://example.com/api/items", {"q":"abc"}, Пусто, {"X-Token":"123"});
  // basic auth:
  r = HTTPRequest("GET", "https://example.com/api/items", Пусто, Пусто, Пусто, "user:pass");
  // или так:
  r = HTTPRequest("GET", "https://example.com/api/items", Пусто, Пусто, Пусто, {"user":"u","pass":"p"});

Результат:

.. code-block:: JavaScript
  
 {
    "ok": true/false,
    "status": <код или 0>,
    "headers": { ... },
    "text": "<тело>",
    "json": <JSONObject/JSONArray или null>
  }

Примечание
""""""""""""

HTTPRequest блокирует поток — на Android запускайте скрипт в фоне (не в UI thread).
