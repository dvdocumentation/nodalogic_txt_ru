.. NodaLogic documentation master file, created by
   sphinx-quickstart on Wed Nov  5 07:29:33 2025.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.
Онлайн обработчик событий HTTPRequest (мобильный клиент)
=============================================================


HTTPRequest - это обработчик события, который отправляет POST-запрос во внешний backend.
Backend получает JSON с данными события, выполняет свою логику и возвращает JSON.
В ответе можно вернуть новые данные и/или команды для UI.

В событии надо выбрать HTTPRequest и в поле function name(method) указать метод запроса. Базу URL платформа возьмет из настроек приложения (секция Онлайн-обработчики), прибавит /<function_name> и пошлет в виде POST-запроса


При этом в запросе отправляется JSON такого вида

.. code-block:: JSON
 
 
 {
   "_data": {
    "...": "текущие данные узла"
  },
  "input_data": {
    "...": "аргументы метода(в некоторых случаях)"
  }
 }


В случае общих событий - только _data

.. code-block:: JSON
 
 
 {
  "_data": {
    "...": "данные события"
  }
 }


На стороне сервера нужно подготовить ответ. Можно повлиять на _data узла, поместив или удалив что то в _data, оно перезапишется. И также можно поместить команды, которые должен выполнить клиент, при возврате ответа.


Пример возврата - добавили status и команды :

.. code-block:: JSON
 
 
 {
  "_data": {
    "_id": "config_uid$Goods$123",
    "name": "Товар 001",
    "barcode": "4601234567890",
    "qty": 1,
    "status": "checked"
  },
  "_commands": [
    {
      "command": "SetTitle",
      "argument": ["Товар проверен"]
    },
    {
      "command": "Refresh"
    },
    {
      "command": "message",
      "argument": ["Штрихкод принят"]
    }
  ]
 }

Важно:
- если вернуть _data в событии узла, клиент заменит node._data;
- если класс autosaved=true, узел будет сохранен;
- если _data не вернуть, данные узла не меняются;
- _commands выполняются после получения ответа;
- argument лучше всегда передавать JSON-массивом.


Формат команды

.. code-block:: JSON
 
 
 {
  "command": "ИмяКоманды",
  "argument": ["аргумент1", "аргумент2"]
 }

Если аргументов нет:

 {
  "command": "Refresh"
 }

Рекомендации:

 * Возвращать всегда JSON-объект.
 * Для бизнес-ошибок лучше возвращать message, а не HTTP 500.
 * _data возвращать только если действительно нужно изменить данные.
 * argument передавать массивом даже для одного аргумента.
 * Имена команд чувствительны к регистру.


6. UI-команды HTTPRequest
-------------------------

Show
~~~~~~
Назначение: показать layout текущего узла.
Формат argument: ``[ layout_array ]``
Пример:
.. code-block:: JSON
 
 
 {
  "command": "Show",
  "argument": [
    {"type":"Text", "text":"Готово"}
  ]
 }

PlugIn
~~~~~~~~~

Назначение: подставить layout/plugin в текущий узел.
Формат argument: [ layout_array ]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "PlugIn",
  "argument": [
    {"type":"Text", "text":"Дополнительный блок"}
  ]
 }

UpdateView
~~~~~~~~~~~~

Назначение: обновить элемент интерфейса.
Формат argument для текущего узла: [view_id, value_json]
Формат argument с node_id: [node_id, view_id, value_json]
Пример:

.. code-block:: JSON
 
 {
  "command": "UpdateView",
  "argument": ["status_text", {"text":"Готово"}]
 }

SetTitle
~~~~~~~~~~~

Назначение: изменить заголовок экрана.
Формат argument: [title]
Пример:
{
  "command": "SetTitle",
  "argument": ["Приемка"]
}

Refresh
~~~~~~~~~~~~~
Назначение: обновить текущую форму узла.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "Refresh"
 }

RefreshTab
~~~~~~~~~~~~
Назначение: обновить вкладку.
Формат argument: [] или [tab_id]
Пример:

.. code-block:: JSON
 
 
 {
  "command": "RefreshTab",
  "argument": ["goods"]
 }

CloseNode
~~~~~~~~~~
Назначение: закрыть текущую форму узла.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "CloseNode"
 }

ScanBarcode
~~~~~~~~~~~~
Назначение: запустить сканирование штрихкода.
Формат argument: [listener, event]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "ScanBarcode",
  "argument": ["onBarcode", "scan"]
 }

Dialog
~~~~~~~~~~~
Назначение: показать диалог.
Формат argument: [title, text, ok_button, cancel_button]
Расширенный формат: [title, text, ok_button, cancel_button, layout_json]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "Dialog",
  "argument": ["Готово", "Операция выполнена", "OK", ""]
 }

AddTimer
~~~~~~~~~~~
Назначение: добавить клиентский таймер.
Формат argument: [timer_key, seconds]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "AddTimer",
  "argument": ["refresh_status", 5]
 }

StopTimer
~~~~~~~~~~
Назначение: остановить клиентский таймер.
Формат argument: [timer_key]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "StopTimer",
  "argument": ["refresh_status"]
 }

StopAllTimers
~~~~~~~~~~~~~~~
Назначение: остановить все клиентские таймеры.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "StopAllTimers"
 }

ShowProgressButton
~~~~~~~~~~~~~~~~~~~
Назначение: показать прогресс на кнопке.
Формат argument: [button_id]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "ShowProgressButton",
  "argument": ["save_button"]
 }

ShowProgressGlobal
~~~~~~~~~~~~~~~~~~~~~~
Назначение: показать глобальный индикатор прогресса.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "ShowProgressGlobal"
 }

HideProgressGlobal
~~~~~~~~~~~~~~~~~~~
Назначение: скрыть глобальный индикатор прогресса.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "HideProgressGlobal"
 }

RunPython
~~~~~~~~~~
Назначение: запустить метод текущего узла. Также поддерживается старое ошибочное имя RunPyhon.
Формат argument: [method_name]
Пример:
.. code-block:: JSON
 
 
 {
  "command": "RunPython",
  "argument": ["onAccept"]
 }

RunEvent
~~~~~~~~~~~

Назначение: запустить событие.
Для узла argument: [event, listener]
Для common argument: [listener, event, parameter2]
Пример для узла:
.. code-block:: JSON
 
 
 
 {
  "command": "RunEvent",
  "argument": ["onInput", "after_http"]
 }

Save
~~~~~~~~~

Назначение: сохранить текущий узел.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "Save"
 }

Upload
~~~~~~~~
Назначение: выгрузить текущий узел на сервер.
Формат argument: []
Пример:
.. code-block:: JSON
 
 
 {
  "command": "Upload"
 }


 Общие UI-команды
~~~~~~~~~~~~~~~~~~~~~

 **toast**
 Назначение: короткое системное toast-сообщение.
 Формат argument: [text]
 Пример:
 .. code-block:: JSON
 
  {
   "command": "toast",
   "argument": ["Сохранено"]
  }

 **message**
 Назначение: snackbar-сообщение внизу экрана.
 Формат argument: [text]
 Пример:
 .. code-block:: JSON
 
 {
   "command": "message",
   "argument": ["Операция выполнена"]
 }

 **vibrate**
 Назначение: вибрация.
 Формат argument: [] или [milliseconds]
 Пример:
.. code-block:: JSON
 
  {
   "command": "vibrate",
   "argument": [300]
  }

 **SendNotification**
 Назначение: показать Android-уведомление.
 Форматы argument:
 - [message]
 - [message, title]
 - [message, title, number, progress]
 Пример:
 .. code-block:: JSON
 
 {
   "command": "SendNotification",
   "argument": ["Задача завершена", "NodaLogic"]
 }

 **SendProgressNotification**
 Назначение: показать/обновить уведомление с прогрессом.
 Формат argument: [message, title, number, progress]
 Пример:
.. code-block:: JSON
 
  {
   "command": "SendProgressNotification",
   "argument": ["Загрузка", "NodaLogic", 1001, 45]
 }

 **beep**
 Назначение: звуковой сигнал.
 Форматы argument:
 - []
 - [tone]
 - [tone, duration_ms, volume]
 Пример:
.. code-block:: JSON
 
  {
   "command": "beep"
  }

 **speak**
 Назначение: произнести текст через Text-To-Speech.
 Формат argument: [text]
 Пример:
 .. code-block:: JSON
 
 {
   "command": "speak",
   "argument": ["Готово"]
 }

 **listen**
 Назначение: запустить распознавание речи, если есть разрешение RECORD_AUDIO.
 Формат argument: []
 Пример:
.. code-block:: JSON
 
  {
   "command": "listen"
  }

 **share_text**

 Назначение: открыть системный share-dialog для текста.
 Формат argument: [text]
 Пример:
.. code-block:: JSON
 
  {
   "command": "share_text",
   "argument": ["Текст для отправки"]
 }

 _SendNotificationProgress
 Назначение: внутренний вариант progress notification. Обычно лучше использовать SendProgressNotification.
 Формат argument: [message, title, number, progress]
 Пример:
.. code-block:: JSON
 
  {
   "command": "_SendNotificationProgress",
   "argument": ["Загрузка", "NodaLogic", 1001, 70]
 }
