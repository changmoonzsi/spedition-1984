Control Center — Архитектурная Библия
Как пользоваться этим документом:
Скопируй его целиком в новый чат с AI. Первые 30 строк — быстрый контекст для AI. Дальше — детали по разделам. В конце — «что приложить к запросу» (какие файлы кода и когда).

Обновляй документ когда: меняется архитектура, добавляется новая сущность, закрывается важное решение. Храни в Notion / GitHub / локально как ARCHITECTURE.md.

0. TL;DR — быстрый контекст для AI (читай первым)
Кто: вайб-кодер, не программист. Пишет с помощью AI (Claude / GPT). Держит логистический отдел.

Что делает: перевозки грузов между Европой / Азией / РФ / СНГ. Ищет машины (перевозчиков) под грузы через WhatsApp-рассылки.

Команда: 14 человек в отделе. У каждого свои регионы, свои перевозчики, свои грузы.

Текущее состояние: две Google-таблицы + два bound Apps Script:

SS «Грузы» — листы ТЛЦ-СНГ РЕФ, Европа (полное плечо) ТЕНТ, Европа (полное плечо) РЕФ. Грузы, которые надо закрыть машинами.

SS «Рассылка» — лист Чистовик. База перевозчиков с фильтрами (страна, направления, типы авто, ADR и т.д.).

GAS-скрипт Обработка — сайдбар с грузами, генерация текста через DeepSeek, кнопка «в рассылку».

GAS-скрипт Рассылка — сайдбар с перевозчиками, фильтры, шаблоны, кнопка «Отправить в WhatsApp».

Мост между таблицами — одна ячейка _BROADCAST_PENDING!A1 (JSON).

Главная боль: устал от сайдбаров (тесно), от моста через одну ячейку (конфликты), нет персональных сохранений для 14 человек.

Целевая архитектура: Node backend + веб-приложение. GAS остаётся только для onOpen (меню в таблице) + управления именными FilterView через scripts.run.

Главный принцип: данные остаются в Google Sheets. Backend — «мозг» (DeepSeek, логика, кеш, профили). Web App — единый UX. GAS — «руки» для того, что требует контекста Sheets.

1. Бизнес-контекст
1.1 Кто пользователь
Отдел логистики, 14 человек

Работают с грузами: ищут перевозчиков через WhatsApp

У каждого — свои регионы и свои перевозчики:

Кто-то только Европа

Кто-то только Китай

Кто-то только СНГ

Кто-то универсал

Рассылка = отправка текста объявления о грузе в WhatsApp перевозчикам, которые подходят под маршрут

1.2 Что делает логист (сценарий)
Открывает приложение

Видит список грузов только своего региона (личный фильтр)

Выбирает груз → система генерирует текст объявления (через DeepSeek)

Нажимает «Отправить в рассылку» → переключается на список перевозчиков

Видит только подходящих под этот груз перевозчиков (фильтр по маршруту)

Копирует текст + номер → вставляет в WhatsApp (пока без API)

Возвращается к грузам, берёт следующий

1.3 Бизнес-сущности
Груз (load) — «Москва → Минск, стройматериалы, 20т, тент, 15.10»

Перевозчик (carrier) — «ИП Иванов, +7..., Беларусь, работает по РФ, тент/реф»

Объявление (broadcast text) — сгенерированный текст для WhatsApp

Рассылка (broadcast) — действие «отправить объявление N перевозчикам»

Профиль фильтров (profile) — сохранённый набор фильтров для конкретного человека

2. Текущее состояние (что работает СЕЙЧАС)
2.1 Две Google-таблицы
Таблица	Назначение	Ключевые листы	ID в .env
SS «Грузы»	Грузы под закрытие	ТЛЦ-СНГ РЕФ, Европа (полное плечо) ТЕНТ, Европа (полное плечо) РЕФ	LOADS_SS_ID
SS «Рассылка»	База перевозчиков	Чистовик, _BROADCAST_PENDING (скрыт)	BROADCAST_SS_ID
2.2 Структура листов в SS «Грузы»
Колонки (14 штук, LOADS_COLS = 14):

Сотрудник (кто взял груз в работу)

Номер заявки

Дата 2

Маршрут (например «Москва - Минск»)

Груз (описание)

Тип перевозки (перецепка / перегрузка / прямая)

Дата

Исходная ставка

…

…

…

Перевозчик (carrierCol) — здесь появляется имя, когда груз закрыт

…

…

Конфиг листов:

javascript
SHEET_CONFIG = {
  'ТЛЦ-СНГ РЕФ':                    { carrierCol: 12, type: 'ref'  },
  'Европа (полное плечо) ТЕНТ + ТЛЦ-СНГ ТЕНТ': { carrierCol: 10, type: 'tent' },
  'Европа (полное плечо) РЕФ':       { carrierCol: 12, type: 'ref'  }
}
Строка 1 — заголовки, строка 2 — начало данных (LOADS_START_ROW = 2).

Цвета: зелёный / жёлтый / красный = строка закрыта или в работе. Белый = свободный груз.

2.3 Структура листа Чистовик (перевозчики)
18 колонок (DATA_COLS = 18), ключевые индексы:

text
0  Автор          (кто добавил перевозчика)
1  Страна         (Belarus, Russia, Turkey, ...)
2  Название       (название компании)
3  Контакты       (имя + телефон + мессенджер)
4  Откуда ЕС      (мульти: 'Германия,Польша')
5  Куда ЕС        (мульти)
6  Азия           (мульти: 'Китай,Киргизия')
7  Куда           (мульти)
8  Откуда ТЛЦ/СНГ (мульти)
9  Куда РФ        (мульти)
10 Типы авто      (мульти: 'Тент,Реф,Контейнеровоз')
11 ADR            (да / нет)
12 Кол-во         (сколько машин)
13 Перецепки/прямой
14 Таможенный
15 Ферреро        (да / нет / частичное — сверка с «Ферреро ПОЛНЫЙ»)
16 Очередь рассылки
17 Комментарий
Мульти-значения разделены запятыми — важно для фильтрации.

2.4 Что уже работает в GAS (bound-скрипты)
Code.gs - Обработка
onOpen — меню «AI Обработка» → «Грузы без перевозчика»

showLoadsSidebar — открывает сайдбар

getActiveRowSnapshot(sheetName, rowNumber) — берёт данные о грузе из строки, проверяет что клик валидный (не заголовок, не закрытая цветом строка, не служебная)

isNonLoadRow(row) — отсекает служебные строки (маркеры: «перевозчик и цена», «забронировано», и т.п.)

classifyBg(hex) — определяет цвет ячейки (white / green / yellow / red / gray)

generateLoadTexts(loads, sheetType) — генерирует 4-строчные тексты через DeepSeek (deepseek-chat, response_format: json_object)

prepareBroadcast(load, generatedText) — записывает JSON payload в _BROADCAST_PENDING!A1 в SS «Рассылка»

getRouteHistory(load) — ищет в других листах «Грузы» прошлые перевозки по тому же маршруту (DeepSeek извлекает страны, потом сверка)

analyzeRouteForBroadcast — DeepSeek подбирает фильтры для рассылки (from_eu, to_eu, asia, to, from_tlc, to_rf, types)

DeepSeek API key хранится в PropertiesService.getScriptProperties() — DEEPSEEK_API_KEY.

Code.gs - Рассылка
onOpen — меню «WhatsApp»

showSidebarWithPending — сайдбар с опциональным payload из обработки

getInitialData(forceReload) — читает лист Чистовик, кеширует (CacheService, TTL 300с)

getOrCreateUserFilterView() — создаёт персональный FilterView через Sheets.Spreadsheets.batchUpdate

Название = WA: <email>

Session.getActiveUser().getEmail() — кто запустил

ID сохраняется в UserProperties (wa_filter_view_id)

updateUserFilterView(filterState) — обновляет критерии FilterView

focusRow(sheetRow) — переход к строке в таблице

savePreset / deletePreset / getPresets — пресеты фильтров в UserProperties

saveTemplate / getTemplateForUser — персональный шаблон сообщения

ferreroSyncCheck — сверка с листом «Ферреро ПОЛНЫЙ» (кое-где стоит «Да», но фактически нет данных)

pollPendingBroadcast() — читает _BROADCAST_PENDING!A1 (для авто-приёма payload)

Sidebar.html - Рассылка
UI: фильтры (14 полей), пресеты, шаблон сообщения, карточка перевозчика, кнопка «Отправить в WhatsApp»

Polling каждые 3 секунды — ищет новый payload из «Обработки»

При получении payload: подставляет текст, применяет фильтры

При клике «Отправить»: window.open('https://wa.me/<phone>?text=<encoded>')

LoadsSidebar.html - Обработка
UI: карточка груза, сгенерированный текст, поле ставки, кнопка «Предложить ставку», «В рассылку», «История маршрута»

Polling через getActiveRowSnapshot — при клике на новую строку генерирует новый текст

prepareBroadcast → потом window.open на SS «Рассылка»

2.5 Что НЕ работает / болит
Проблема	Симптом
Мост через _BROADCAST_PENDING!A1	Один клик перезаписывает другой. Конфликт при 2+ юзерах
Сайдбар 430px	Тесно для таблиц с 18 колонками
UserProperties привязаны к аккаунту	Сменил устройство — потерял пресеты
getRouteHistory медленный	DeepSeek вызывается на каждое сравнение маршрутов
Все редактируют одну таблицу	Каждый видит чужие FilterView (названия «WA: Иван», «WA: Петя»)
3. Целевая архитектура
3.1 Схема верхнего уровня
text
┌────────────────────────────────────────────────────────┐
│  Web App (React + Vite)                                │
│  • Список грузов с фильтрами по профилю                │
│  • Карточка груза + генерация текста                   │
│  • Кнопка «В рассылку» → список подходящих перевозчиков│
│  • Копирование в WhatsApp Web (ручное)                 │
└────────────────────┬───────────────────────────────────┘
                     │ HTTPS
┌────────────────────▼───────────────────────────────────┐
│  Backend (Node.js + Fastify)                           │
│  • OAuth2 refresh-token (ОДИН сервисный аккаунт)       │
│  • Sheets API — чтение/запись в обе SS напрямую        │
│  • Scripts API — только FilterView через scripts.run   │
│  • DeepSeek API (ключ в .env)                          │
│  • SQLite — профили, шаблоны, логи                     │
│  • In-memory кеш (lru-cache, TTL 15-30с)               │
└────────────────────┬───────────────────────────────────┘
                     │
     ┌───────────────┴───────────────┐
     ▼                               ▼
┌──────────────────┐          ┌──────────────────┐
│ SS «Грузы»       │          │ SS «Рассылка»    │
│ (bound GAS)      │          │ (bound GAS)      │
│ onOpen меню +    │          │ onOpen меню +    │
│ FilterView через │          │ FilterView через │
│ scripts.run      │          │ scripts.run      │
└──────────────────┘          └──────────────────┘
3.2 Разделение ответственности
Слой	Что делает	Чего НЕ делает
Web App	UX, фильтры, показ данных, копирование, кнопки действий	Не хранит секреты, не ходит в Google напрямую
Backend	Вся бизнес-логика, генерация, история, профили, кеш	Не рисует UI
GAS	onOpen меню + управление FilterView + focusRow	Не генерирует тексты, не хранит данные
3.3 Что переносим из GAS в Node
Функция	Куда	Почему
generateLoadTexts / callDeepSeekJson	Backend	Ключ в .env, легче дебажить, кеширование
prepareBroadcast	Backend (очередь в SQLite)	Не через одну ячейку, а через таблицу-очередь
getRouteHistory	Backend (SQLite + кеш)	Тяжёлое чтение — батчить, кешировать
ferreroSyncCheck	Backend	Чистая логика
getInitialData	Backend	Кеш + пагинация
savePreset / saveTemplate	Backend + SQLite	UserProperties теряются
getActiveRowSnapshot	Удаляем	Заменяется списком грузов в Web App
pollPendingBroadcast	Удаляем	Backend сам пишет и читает очередь
3.4 Что ОСТАЁТСЯ в GAS
Функция	Почему
onOpen (обе SS)	Меню в таблице, нельзя вызвать извне
getOrCreateUserFilterView	Требует Sheets context (Sheets.Spreadsheets.batchUpdate)
updateUserFilterView	То же
removeUserFilterView	То же
focusRow	Быстрее через batchUpdate из Node, но можно оставить в GAS
Критичное изменение: getOrCreateUserFilterView сейчас использует Session.getActiveUser().getEmail(). При вызове через scripts.run это вернёт email владельца refresh-token (то есть твой), а не того, кто сейчас в веб-приложении. Параметр userName должен приходить из Backend явно.

4. Модель данных
4.1 Google Sheets (без изменений в структуре)
SS «Грузы» — 3 листа грузов, 14 колонок, 3 листа конфига.
SS «Рассылка» — Чистовик (18 колонок) + _BROADCAST_QUEUE (новая, 7 колонок, скрытая).

Новый лист _BROADCAST_QUEUE (вместо _BROADCAST_PENDING):

Кол	Поле	Тип	Пример
A	id	uuid	bc_20251031_143522_a7f2
B	user	string	Иван
C	createdAt	ISO date	2025-10-31T14:35:22Z
D	loadSheet	string	ТЛЦ-СНГ РЕФ
E	loadRow	int	42
F	text	string	«Нужен тент на 15.10…»
G	filters	JSON	{"country":["Belarus"],"to_rf":["Москва"]}
H	status	enum	pending / read / done
Почему очередь, а не одна ячейка: каждый клик = новая строка, не перезаписывает предыдущий. 14 человек могут нажать одновременно — не мешают друг другу.

4.2 SQLite (Backend)
sql
-- Пользователи (простой вход по имени, без OAuth)
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Профили фильтров
CREATE TABLE profiles (
  id INTEGER PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  name TEXT NOT NULL,           -- 'Москва-Европа'
  region TEXT,                   -- 'europe' | 'asia' | 'cis' | 'all'
  filters_json TEXT NOT NULL,    -- полный JSON фильтров
  is_default INTEGER DEFAULT 0,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Шаблоны сообщений
CREATE TABLE templates (
  id INTEGER PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  name TEXT,
  body TEXT NOT NULL,
  is_default INTEGER DEFAULT 0,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- История рассылок (лог)
CREATE TABLE broadcasts (
  id TEXT PRIMARY KEY,           -- uuid
  user_id INTEGER REFERENCES users(id),
  load_sheet TEXT,
  load_row INTEGER,
  load_snapshot_json TEXT,       -- снимок груза на момент отправки
  text TEXT,
  filters_json TEXT,
  carriers_json TEXT,            -- список перевозчиков, кому отправили
  status TEXT DEFAULT 'pending', -- pending | sent | failed
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  completed_at TEXT
);

-- Кеш истории маршрутов (ускоряет getRouteHistory)
CREATE TABLE route_history (
  id INTEGER PRIMARY KEY,
  route_normalized TEXT NOT NULL,
  from_country TEXT,
  to_country TEXT,
  carrier TEXT,
  rate TEXT,
  load_sheet TEXT,
  load_row INTEGER,
  raw_json TEXT,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(route_normalized, carrier, load_sheet, load_row)
);
CREATE INDEX idx_route ON route_history(route_normalized);
4.3 Хранилище сессий
Простое: express-session + connect-sqlite3 (сессии в той же SQLite).

Пользователь входит один раз (имя → cookie)

Cookie живёт 90 дней

При первом входе — запись в users

Никакого Google OAuth для 14 человек (это оставляем на потом, вариант B из прошлого обсуждения).

5. API Контракты (Backend — Fastify)
5.1 Auth
Метод	Путь	Тело	Ответ
POST	/api/auth/login	{ name: "Иван" }	{ ok: true, user: { id, name } } + cookie
POST	/api/auth/logout	—	{ ok: true }
5.2 Грузы
Метод	Путь	Параметры	Ответ
GET	/api/loads	?sheet=ТЛЦ-СНГ РЕФ&profile_id=1	{ loads: [...], total, lastModified }
GET	/api/loads/:sheet/:row	—	{ load: {...} }
POST	/api/loads/:sheet/:row/generate	{ }	{ text: "..." } — DeepSeek
POST	/api/loads/:sheet/:row/history	{ }	{ countries, matches: [...] }
5.3 Перевозчики
Метод	Путь	Параметры	Ответ
GET	/api/carriers	?filters=<json>&search=<q>	{ carriers: [...], total }
GET	/api/carriers/filters	—	{ filters: { country: [...], ... } } — уникальные значения для UI
5.4 Рассылка
Метод	Путь	Тело	Ответ
POST	/api/broadcast/prepare	{ loadSheet, loadRow, text, filters }	{ id, url } — запись в _BROADCAST_QUEUE
GET	/api/broadcast/queue	?status=pending	{ items: [...] }
PATCH	/api/broadcast/:id	{ status: "done" }	{ ok: true }
5.5 Профили и шаблоны
Метод	Путь	Тело	Ответ
GET	/api/profiles	—	{ profiles: [...] }
POST	/api/profiles	{ name, filters }	{ profile }
PATCH	/api/profiles/:id	{ name?, filters?, is_default? }	{ profile }
DELETE	/api/profiles/:id	—	{ ok: true }
GET	/api/templates	—	{ templates }
POST	/api/templates	{ name, body }	{ template }
5.6 FilterView (проксирует scripts.run)
Метод	Путь	Тело	Что делает
POST	/api/filterview/apply	{ userName, sheetName, filters }	Вызывает scripts.run(getOrCreateUserFilterView, [userName]) + updateUserFilterView(userName, filters)
DELETE	/api/filterview/:userName	—	scripts.run(removeUserFilterView, [userName])
5.7 Служебное
Метод	Путь	Ответ
GET	/api/health	{ ok: true, uptime, version }
GET	/api/status	{ sheets: { loads: "ok", broadcast: "ok" }, deepseek: "ok" }
6. GAS-функции — что меняется
6.1 getOrCreateUserFilterView(userName) — новая сигнатура
javascript
// БЫЛО: const email = Session.getActiveUser().getEmail() || 'anon';
// СТАЛО:
function getOrCreateUserFilterView(userName) {
  // при вызове из scripts.run Session вернёт email владельца токена
  // поэтому имя приходит явно
  var name = String(userName || '').trim() || 'unknown';
  var title = 'WA: ' + name;
  // ... остальное как сейчас, только title = 'WA: ' + name
}
Почему так: все 14 FilterView будут созданы от имени одного сервисного аккаунта (твой). Они все видны в меню «Данные → Режим фильтрации» с разными названиями. Каждый логист открывает «WA: Иван» — видит только свой. Случайно кликнуть в чужой — увидит ту же таблицу, что и так видит. Не утечка.

6.2 Backend вызывает GAS
javascript
// backend/services/gas.js
const { google } = require('googleapis');
const script = google.script({ version: 'v1', auth });

async function callGasFn(scriptId, fnName, params = []) {
  const res = await script.scripts.run({
    scriptId,
    requestBody: { function: fnName, parameters: params, devMode: true }
  });
  if (res.data.error) throw new Error(JSON.stringify(res.data.error));
  return res.data.response?.result;
}

module.exports = { callGasFn };
Использование:

javascript
await callGasFn(process.env.BROADCAST_SCRIPT_ID, 'getOrCreateUserFilterView', ['Иван']);
await callGasFn(process.env.BROADCAST_SCRIPT_ID, 'updateUserFilterView', ['Иван', filtersJson]);
6.3 Что удаляем из GAS
pollPendingBroadcast — не нужно, backend сам пишет в очередь

getActiveRowSnapshot — заменён эндпоинтом /api/loads/:sheet/:row

generateLoadTexts — переехал в Node

prepareBroadcast — переехал в Node

getRouteHistory — переехал в Node

savePreset / saveTemplate — в SQLite

6.4 onOpen — оставляем как есть
Меню в Sheets: «WhatsApp» → «Открыть панель рассылки» (открывает Web App в новой вкладке: window.open(WEBAPP_URL)).

7. Ключевые решения и обоснования (ADR)
ADR-001: Node backend + Web App вместо только GAS
Решение: переезд на Node + React.

Почему:

Сайдбар 430px тесен для 14 человек и 18 колонок

UserProperties привязаны к браузеру — не работают между устройствами

14 человек в одной таблице = конфликты, каждый видит чужие FilterView

Ключи DeepSeek / Google — в GAS неудобно, в .env лучше

Тестировать GAS больно, Node проще

Цена: ~4 недели работы, нужен сервер (Railway / Render / VPS).

ADR-002: FilterView остаётся, но именной
Решение: каждая View называется WA: <userName>, создаётся через scripts.run.

Почему: альтернатива (OAuth на 14 человек) — +1.5 недели и боль с согласиями. Текущая схема работает «достаточно хорошо» — каждый открывает свою.

Когда пересмотреть: если кто-то реально будет путаться в чужих View (через 2-3 месяца после запуска).

ADR-003: Очередь рассылки вместо одной ячейки
Решение: лист _BROADCAST_QUEUE (строки) вместо _BROADCAST_PENDING!A1 (одна ячейка).

Почему: 14 человек одновременно нажимают «В рассылку» → сейчас второй затирает первого. С очередью — каждый пишет свою строку, ничего не теряется.

ADR-004: SQLite, а не PostgreSQL
Решение: SQLite (файл на сервере).

Почему: 14 пользователей, несколько сотен записей в день. SQLite справится с огромным запасом. Postgres — оверкилл на старте. Если упрёмся — миграция несложная.

ADR-005: Простой вход по имени, а не Google OAuth
Решение: при первом открытии Web App — ввод имени, cookie на 90 дней.

Почему: Google OAuth для 14 человек = 14 согласий, отзывы токенов, consent screen. Простой вход = 1 форма, никаких разрешений. Не безопасно с точки зрения аутентификации, но у нас не банк — отдел логистики в закрытой сети.

Когда пересмотреть: когда добавим реальную отправку в WhatsApp (там нужна ответственность за действия).

ADR-006: In-memory кеш (lru-cache), а не Redis
Решение: lru-cache в Node.

Почему: 14 пользователей, один инстанс. Redis = лишний сервис. In-memory хватит.

Когда пересмотреть: при переходе на несколько Node-инстансов (горизонтальное масштабирование).

ADR-007: Ручное копирование в WhatsApp, а не API
Решение: на MVP — кнопка «Скопировать текст» + window.open('https://wa.me/...').

Почему: WhatsApp Business API — отдельная интеграция, верификация, деньги. Пока хватает ручного копирования.

Когда добавить: если ручное копирование реально станет узким местом.

8. Ловушки и как их избегать
8.1 onEdit / onChange не срабатывают на записи через API
sheets.spreadsheets.values.update из Node не запускает onEdit в bound-скрипте.

Как проверить: аудит обеих SS — найти все onEdit / onChange.

Что делать:

Если логика некритична → портировать в Node

Если критична → писать через scripts.run (медленнее, 1-2 сек, но сохраняет поведение)

8.2 Session.getActiveUser() в scripts.run возвращает владельца токена
См. ADR-002 и раздел 6.1. Всегда передавай userName явно.

8.3 Два человека одновременно ставят машину на один груз
14 человек = это будет каждый день.

Решение (с первого дня):

При записи «перевозчик на груз» backend читает текущее значение колонки carrierCol

Если уже занято → 409 Conflict → UI: «Обновите, груз только что закрыли»

Автообновление списка раз в 15 сек (polling modifiedTime)

8.4 DeepSeek rate limits
DeepSeek API имеет лимиты (запросов в минуту / токенов).

Решение:

Кеш сгенерированных текстов в SQLite (hash(sheet,row,text_template) → text)

При повторном клике на тот же груз — берём из кеша

Retry с exponential backoff на 429

8.5 Квоты Google Sheets API
~300 запросов/мин на проект.

Решение:

Батчить чтения (values.batchGet по списку диапазонов)

Кешировать с TTL 15-30 сек

Не читать строку по одной

8.6 Свежесть данных (кто-то редактирует в Sheets руками)
Sheets API не имеет push-уведомлений на изменение данных.

Решение:

Polling drive.files.get(modifiedTime) раз в 15 сек

Если изменился — инвалидация кеша и перечитывание нужных диапазонов

UI: «Обновлено 5 секунд назад»

8.7 Синтаксис .env ломается незаметно
Особенно на Windows: .env.txt, BOM, кодировка.

Решение:

Проверять длины: Script ID 57-60 символов, Spreadsheet ID 44

test-run.js как healthcheck

8.8 Деплой GAS через API Executable
Настройки (для каждого bound-скрипта):

Apps Script → ⚙️ Настройки проекта → привязать к GCP-проекту

Развернуть → Новое развёртывание → API Executable (не Web App!)

Execute as: Me

Скопировать Script ID (не Deployment ID!), длина 57-60, начинается с 1

9. Roadmap
Фаза 0 — Подготовка (уже сделано)
☑ GCP-проект
☑ Привязка bound-скриптов к GCP
☑ API Executable deployment
☑ OAuth refresh-token (Desktop app)
☑ scripts.run работает (проверено test-run.js)
☑ Consent screen → In production
☑ .env заполнен и проверен
Фаза 1 — Скелет Backend + Web App (Неделя 1)
□ Node backend скелет (Fastify, health check, .env, логгер)
□ SQLite схема (users, profiles, templates, broadcasts, route_history)
□ Google OAuth client в Node (refresh-token из .env)
□ GET /api/loads — чтение листов грузов, список
□ GET /api/carriers — чтение Чистовик, фильтры
□ Простой UI (React + Vite): список грузов + карточка + список перевозчиков
□ Авторизация по имени + cookie
Фаза 2 — Генерация и очередь рассылки (Неделя 2)
□ POST /api/loads/:sheet/:row/generate → DeepSeek
□ Кеш генерации в SQLite
□ Лист _BROADCAST_QUEUE в SS «Рассылка»
□ POST /api/broadcast/prepare → запись в очередь
□ GET /api/broadcast/queue → список своих pending
□ Отключить старый сайдбар у 2-3 пилотов
Фаза 3 — FilterView через scripts.run (Неделя 3)
□ Переделать getOrCreateUserFilterView(userName) в GAS
□ Переделать updateUserFilterView(userName, filters) в GAS
□ POST /api/filterview/apply (Backend → scripts.run)
□ Проверить: в Sheets появляются «WA: Иван», «WA: Петя»
□ Все 14 человек переходят на Web App
Фаза 4 — История маршрутов и персонификация (Неделя 4)
□ POST /api/loads/:sheet/:row/history → SQLite + кеш
□ Профили фильтров (/api/profiles)
□ Шаблоны сообщений (/api/templates)
□ Автообновление (modifiedTime polling)
□ Проверка конфликта «поставить авто на груз»
□ Удалить старые сайдбары
Фаза 5 — Позже (по потребности)
□ WhatsApp Business API (реальная отправка)
□ Google OAuth на 14 человек (настоящие приватные FilterView)
□ Серверная авторизация с паролями
□ Метрики / дашборд для руководителя
□ Мобильная версия UI
10. Чек-лист «что сделано / что в работе»
Обновляй раздел вручную. Формат:

text
[✅] Готово
[🚧] В работе (укажи ожидаемое завершение)
[⬜] В плане
[❌] Отменено (с причиной)
Пример:

text
[✅] Фаза 0: OAuth refresh-token, API Executable, test-run.js
[🚧] Фаза 1: backend skeleton — 60%, остались carriers endpoint
[⬜] Фаза 1: UI список грузов
[⬜] Фаза 2: DeepSeek endpoint
[❌] Старый pollPendingBroadcast — удалён (заменён на queue)
11. Переменные окружения (.env)
env
# OAuth-клиент (Desktop app из GCP Console)
GOOGLE_CLIENT_ID=XXXXXXXXXX-YYYYYYY.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-XXXXXXXXXXXXXX
GOOGLE_REFRESH_TOKEN=1//0gXXXXXXXX...

# Script ID (НЕ Deployment ID!) — Apps Script → ⚙️ → Идентификаторы
LOADS_SCRIPT_ID=1....(57-60 символов)
BROADCAST_SCRIPT_ID=1....(57-60 символов)

# Spreadsheet ID — из URL между /d/ и /edit, длина 44
LOADS_SS_ID=1....(44 символа)
BROADCAST_SS_ID=1oSbNLUfLdOcBl_0O83YxPqLAX8VXVNSBlJH3MreEEZM

# DeepSeek
DEEPSEEK_API_KEY=sk-...
DEEPSEEK_MODEL=deepseek-chat

# Server
PORT=3000
WEBAPP_URL=http://localhost:5173
NODE_ENV=development
SESSION_SECRET=random-32-chars
SQLITE_PATH=./data/control-center.db

# Polling
MODIFIED_TIME_POLL_MS=15000
Проверка длин (PowerShell):

powershell
Get-Content .env | Where-Object { $_ -match "_ID=" } | ForEach-Object {
  $p = $_ -split "=", 2
  "$($p[0].Trim()): длина=$($p[1].Length), начало=$($p[1].Substring(0,4))"
}
Script ID → 57-60, начало 1A / 1d / 14

Spreadsheet ID → 44

Deployment ID (не используется!) → AKfycb

12. Как прикладывать к запросу в новый AI-чат
Минимальный пакет (архитектурные вопросы)
Этот документ целиком

Для правки Backend
Этот документ

Файлы из backend/: server.js, routes/*.js, services/*.js, db/schema.sql

Для правки GAS
Этот документ

Code.gs (обработка), Code.gs (рассылка), LoadsSidebar.html, Sidebar.html

Для правки Frontend
Этот документ

Файлы из frontend/src/: App.jsx, pages/*, api/*

Для правки специфичной фичи
Этот документ

Только файлы, касающиеся фичи

+ одна строка контекста: «Сейчас работаю над X, вот проблема: Y, вот код: Z»

Правило одной фразы в начале любого AI-запроса
«Это проект Control Center (логистика, 14 человек, Node + GAS + Google Sheets). Архитектура в документе. Сейчас делаю [X]. Вот код: [Y]. Проблема / вопрос: [Z].»

Так любой AI в первом абзаце понимает: масштаб, стек, цель. Дальше — конкретика.

13. Глоссарий
Термин	Что это
SS	Spreadsheet (Google-таблица)
GAS	Google Apps Script
Bound script	Скрипт, привязанный к конкретной таблице
FilterView	Именованный фильтр в Google Sheets (меню «Данные → Режим фильтрации»)
scripts.run	API-вызов GAS-функции из Node
Payload	JSON-сообщение с данными о грузе и фильтрах для рассылки
Carrier	Перевозчик (компания / ИП с машинами)
Load	Груз (заявка на перевозку)
Profile	Сохранённый набор фильтров для конкретного человека
ADR	Architecture Decision Record — документированное решение с обоснованием
14. Контакты и владение
Владелец проекта: ты (вайб-кодер, не программист)

Разработчик: ты + AI (Claude / GPT)

Пользователи: 14 человек отдела логистики

Хостинг (план): Railway / Render / VPS (Node + SQLite)

Резервное копирование: SQLite → ежедневный бэкап на Google Drive

15. История изменений документа
Дата	Что изменилось
2025-10-31	Создан документ. Зафиксирован переезд с GAS-сайдбаров на Node + Web App. ADR-001 … ADR-007.
Всё. Сохрани этот документ как ARCHITECTURE.md. Обновляй раздел 10 (чек-лист) и раздел 15 (история) по мере продвижения. При любом новом AI-запросе — копируй целиком в начало.

Если хочешь — распишу следующим шагом:

Скелет проекта (backend/ + frontend/ структура папок, package.json, server.js скелет)

Или SQLite миграции (полные schema.sql)

Или конкретный первый эндпоинт /api/loads целиком (код)
