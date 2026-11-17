# Отчёт по лабораторной работе №2. Node-RED

Студентка: Коростелкина Диана Алексеевна, группа СДП-УИР-241.

## 1. Краткое описание выполненного

В лабораторной работе я развернула Node-RED в Docker, собрала 12 потоков (flow) с базовыми и расширенными нодами и выполнила одну ачивку. Потоки охватывают: вывод сообщений, обработку данных (function, switch, change, template), запросы к внешнему API, обмен сообщениями по MQTT через публичный брокер, собственные REST-эндпоинты, dashboard с gauge и графиком, Telegram-бота, чтение и запись файлов и работу с контекстом. Каждый поток сохранён отдельным файлом `lab2/flows/flow-NN-<имя>.json`, документация по эндпоинтам лежит в [api.md](api.md). Во всех потоках в названиях, `msg.topic` и payload использованы моя фамилия и группа.

## 2. Способ установки, версии

- Способ установки: Docker Desktop (Windows), готовый образ `nodered/node-red`.
- Команда запуска:

```
docker run -d -p 1880:1880 -v C:\node-red-data:/data --name mynodered nodered/node-red
```

- Порт: `1880`, редактор открывается по адресу `http://localhost:1880`.
- Volume: папка `C:\node-red-data` на компьютере связана с папкой `/data` в контейнере. Поэтому потоки, установленные пакеты и файлы, которые создаёт Node-RED, сохраняются после перезапуска контейнера.
- Версия Node-RED: 5.0.9.
- Версия Node.js: `v24.22.2`

![Версия Node.js](../screenshots/00-node-version.png)

![О Node-RED](../screenshots/00-about.png)

Дополнительно установленные пакеты (через ☰ → «Управление палитрой»):

| Пакет | Для чего |
|---|---|
| `node-red-dashboard` 3.6.6 | gauge, chart, форма и другие виджеты dashboard (пункт 2.9, ачивка) |
| `node-red-contrib-telegrambot` | Telegram-бот (пункт 2.10) |

Токен Telegram-бота в репозиторий не попадает: Node-RED хранит его отдельно в файле учётных данных, вне репозитория.

## 3. Какие AI-промпты использовались

ИИ использовался для генерации кода function-нод, шаблона Mustache и flow-файлов. Ключевые примеры:

1. «Напиши код для function node в Node-RED: принимает объект с массивом чисел, считает сумму, среднее и чётные числа, использует let/const, if/else, цикл for, массив и объект, возвращает объект с полем payload.»
2. «Сделай Mustache-шаблон для template node, который собирает JSON из объекта с полями name, group и subject.»
3. «Сделай три GET-эндпоинта в Node-RED: /api/text, /api/info и /api/items с path- и query-параметрами, чтобы при неверных значениях возвращались 400 и 404.»
4. «Как в Node-RED сделать dashboard с gauge и графиком, который имитирует датчик температуры?»
5. «Как создать Telegram-бота через BotFather и подключить его в Node-RED с командами /start и /time и echo-ответом?»
6. «Как сделать форму на dashboard, которая меняет порог и имя топика без редеплоя?»

Весь сгенерированный код я проверяла в Node-RED, а на защите могу объяснить каждую ноду.

## 4. Освоенные ноды

| Категория | Ноды | Где использованы |
|---|---|---|
| common | `inject`, `debug` | все потоки |
| function | `function`, `switch`, `change`, `template` | 2.2 - 2.5, 2.8, 2.11, 2.12 |
| network | `http request`, `http in`, `http response`, `mqtt in`, `mqtt out` | 2.6 - 2.8 |
| storage | `file in`, `file out` | 2.11 |
| dashboard | `ui_gauge`, `ui_chart`, `ui_text`, `ui_form`, `ui_toast`, `ui_tab`, `ui_group` | 2.9, ачивка |
| telegram | `command`, `receiver`, `sender` | 2.10 |

Также освоены: контекст (`flow`, `global`), переменные окружения (`env.get`), отключение и включение потока, импорт и экспорт потоков в JSON.

## 5. Потоки и скриншоты

### 2.1. Inject → Debug
`inject` каждые 5 секунд отправляет строку с фамилией и `msg.topic = lab2/Korostelkina/basic`, `debug` показывает весь объект сообщения. Файл: [flow-01-inject-debug.json](../flows/flow-01-inject-debug.json)

![2.1](../screenshots/01-inject-debug.png)

### 2.2. Function
Функция принимает объект с массивом чисел, в цикле `for` считает сумму и выбирает чётные числа, через `if/else` определяет уровень (`high` или `low`) и возвращает объект. Файл: [flow-02-function.json](../flows/flow-02-function.json)

![2.2](../screenshots/02-function.png)

### 2.3. Switch
`switch` по значению `msg.payload.level` направляет сообщение на один из двух `debug`. Файл: [flow-03-switch.json](../flows/flow-03-switch.json)

![2.3](../screenshots/03-switch.png)

### 2.4. Change
`change` задаёт `msg.topic`, `msg.timestamp` (текущее время, выражение `$now()`) и `msg.payload`. Файл: [flow-04-change.json](../flows/flow-04-change.json)

![2.4](../screenshots/04-change.png)

### 2.5. Template
Mustache-шаблон собирает JSON из полей объекта, который отправляет `inject`. Файл: [flow-05-template.json](../flows/flow-05-template.json)

![2.5](../screenshots/05-template.png)

### 2.6. HTTP request
GET-запрос к публичному API `https://api.chucknorris.io/jokes/random` с разбором ответа в JSON-объект. Сначала я использовала `catfacts.ninja`, но он оказался недоступен (ошибка `ENOTFOUND`). Я проверила, что сеть в контейнере работает (запрос к другому сайту через `docker exec mynodered wget ...` прошёл), и заменила API. Файл: [flow-06-http-request.json](../flows/flow-06-http-request.json)

![2.6](../screenshots/06-http-request.png)

### 2.7. MQTT
Одна ветка публикует случайную температуру каждые 5 секунд в топик `student/Korostelkina/lab2/temperature-SDP-UIR-241` на публичном брокере `broker.hivemq.com:1883`, другая подписана на тот же топик и выводит сообщение в `debug`. Сообщения возвращаются обратно, значит, они прошли через брокер. Файл: [flow-07-mqtt.json](../flows/flow-07-mqtt.json)

![2.7](../screenshots/07-mqtt.png)

### 2.8. GET-эндпоинты
Три эндпоинта: `/api/text` (текст), `/api/info` (JSON с двумя полями), `/api/items` (path-параметр `:id` и query-параметр `limit`, ответы 200, 400 и 404). Подробное описание, параметры и примеры запросов и ответов: [api.md](api.md). Файл: [flow-08-endpoints.json](../flows/flow-08-endpoints.json)

![Поток](../screenshots/08-endpoints-flow.png)
![/api/text](../screenshots/08-text.png)
![/api/info](../screenshots/08-info.png)
![/api/items/2 (успех)](../screenshots/08-items-ok.png)
![/api/items/99 (404)](../screenshots/08-items-404.png)
![/api/items/abc (400)](../screenshots/08-items-400.png)
![/api/items?limit=2](../screenshots/08-items-limit.png)

### 2.9. Dashboard
Имитатор датчика температуры (inject каждые 2 секунды и function) выводит значение на gauge и на график, страница доступна по адресу `http://localhost:1880/ui`. Файл: [flow-09-dashboard.json](../flows/flow-09-dashboard.json)

![Поток](../screenshots/09-dashboard-flow.png)
![Dashboard](../screenshots/09-dashboard-ui.png)

### 2.10. Telegram-бот
Бот `tvl_lab2_Korostelkina_bot`: команда `/start` (приветствие), команда `/time` (текущее время) и echo-ответ на любое другое сообщение. Файл: [flow-10-telegram.json](../flows/flow-10-telegram.json)

![Поток](../screenshots/10-telegram-flow.png)
![Чат](../screenshots/10-telegram-chat.png)

### 2.11. Чтение и запись файла
Запись добавляет CSV-строку в файл `/data/lab2/records.csv`, чтение выводит содержимое файла в `debug`. Файл лежит в volume, поэтому данные сохраняются после `docker restart mynodered`. Файл: [flow-11-files.json](../flows/flow-11-files.json)

![Поток](../screenshots/11-files-flow.png)
![Чтение](../screenshots/11-files-read.png)
![Файл на компьютере](../screenshots/11-files-explorer.png)
![Команда перезапуска](../screenshots/11-files-restart-cmd.png)
![После перезапуска](../screenshots/11-files-after-restart.png)

### 2.12. Контекст
Счётчик в `flow context` живёт между сообщениями, время последнего нажатия хранится в `global context`, имя и группа берутся из переменных окружения потока (`env.get`). Контекст хранится в памяти, поэтому сбрасывается при перезапуске Node-RED. Файл: [flow-12-context.json](../flows/flow-12-context.json)

![Поток](../screenshots/12-context-flow.png)
![Счётчик](../screenshots/12-context-counter.png)
![Контекстные данные](../screenshots/12-context-sidebar.png)
![Переменные окружения](../screenshots/12-env-vars.png)

## 6. Ачивка 6. Dashboard forms

На странице dashboard добавлена форма (`ui_form`) с полями «Порог срабатывания», «Имя топика» и «Максимум генератора». При нажатии «Применить» значения записываются в `flow context`, а генератор читает их при каждом срабатывании. Поэтому изменения применяются сразу, без нажатия «Развернуть» (на скриншоте видно, что кнопка «Развернуть» неактивна). Файл: [flow-13-achievement-dashboard-forms.json](../flows/flow-13-achievement-dashboard-forms.json)

![Поток](../screenshots/13-flow.png)
![До изменения](../screenshots/13-before.png)
![Подтверждение](../screenshots/13-toast.png)
![После изменения](../screenshots/13-after.png)
![Без редеплоя](../screenshots/13-no-redeploy.png)

## 7. Выводы

В ходе лабораторной работы я освоила Node-RED как low-code инструмент: потоки собираются из готовых нод, а код нужен только там, где требуется своя логика (function-ноды и шаблоны). Запуск в Docker с volume оказался удобным: потоки, установленные пакеты и файлы лежат в папке на компьютере и не пропадают при перезапуске контейнера. Я разобралась, как данные проходят по потоку в виде объекта `msg`, как ноды меняют его поля, как работает ветвление (`switch`) и как хранить данные в контексте (`flow`, `global`) и переменных окружения. Я научилась создавать собственные REST-эндпоинты с path- и query-параметрами и возвращать корректные статусы 200, 400 и 404, а также подключать внешние сервисы: публичный API, MQTT-брокер и Telegram-бота. Dashboard показал, как быстро можно сделать интерфейс для данных, а ачивка с формой показала, как менять поведение потока без редеплоя через контекст. Отдельно я столкнулась с недоступностью внешнего API (`catfacts.ninja`), научилась проверять сеть из контейнера и заменять сервис. Каждый поток хранится в JSON-файле, поэтому его удобно версионировать в Git вместе с остальными лабораторными работами.
