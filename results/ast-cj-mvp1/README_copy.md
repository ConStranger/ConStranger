# Склерозник — персональный ИИ-ассистент для заметок и напоминаний

Telegram-бот, который принимает произвольные фразы на русском языке, классифицирует их через локальную LLM (трёхпроходная архитектура), сохраняет в базу данных с извлечением сущностей, тегов и временных меток, и управляет напоминаниями. Работает полностью на локальной инфраструктуре — данные не покидают домашний сервер.

## Возможности

**Сохранение:** отправьте боту произвольную фразу — LLM-классификатор (Pass 1) определит тип записи (reminder, link, fact, note, secret), Pass 2a определит подтип, Pass 2b извлечёт сущности. Бот подтверждает сохранение с id, тегами, сущностями и временем напоминания.

**Поиск:** запросы естественным языком — бот ищет по содержимому, тегам и сущностям.

**Напоминания:** относительное время («через час», «завтра в 10», «в 23 часа») вычисляется библиотекой dateparser, не LLM. Фоновый планировщик проверяет наступившие напоминания каждую минуту.

**Управление записями:**
- `/list [N]` — последние N сохранённых записей, автосплит длинных сообщений
- `/listall [N]` — лог последних N сообщений с маркерами удаления и типами
- `/show <id>` — подробный просмотр записи
- `/fix <id> <новый текст>` — перезапись записи с повторной классификацией
- `/reclassify <id>` — переклассификация записи по существующему тексту
- `/undo` — мягкое удаление последней записи
- `/del <id>` — мягкое удаление записи по id
- `/restore [id]` — восстановление удалённой записи
- `/entity [list]` — управление сущностями
- `/alias <entity_id> <имя>` — добавление алиаса сущности
- Редактирование Telegram-сообщения — автоматически перезаписывает исходную запись

**Просмотр дел:** запрос «какие дела на сегодня» — бот покажет все напоминания на день с маркерами статуса (✅ выполнено, ⏰ просрочено, 🕐 запланировано).

## Технологии

| Компонент | Технология |
|---|---|
| LLM | Ollama, модель `qwen3.5:4b-q4_K_M` |
| Bot framework | python-telegram-bot 21.5 (polling) |
| База данных | SQLite |
| Планировщик | APScheduler 3.10 (AsyncIOScheduler) |
| HTTP клиент | httpx 0.27 |
| Парсинг времени | dateparser |
| Среда выполнения | Python 3.12, venv |

## Архитектура

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Telegram    │────▶│   bot.py     │────▶│  classifier.py  │
│  (пользователь)│◀────│ (обработчик)  │◀────│  (3 прохода)     │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                       │
                    ┌──────▼───────┐        ┌──────▼───────┐
                    │   db.py      │        │   Ollama API  │
                    │  (SQLite)    │        │  (LLM 4B Q4)  │
                    └──────┬───────┘        └──────────────┘
                           │
                    ┌──────▼───────┐
                    │ scheduler.py │
                    │ (APScheduler) │
                    └──────────────┘
```

**Трёхпроходная классификация:**

```
Пользователь → bot.py → classifier.py
                         │
                    ┌────▼─────┐
                    │  Pass 1   │  intent + type + content + url + remind_at + entity_hint
                    │ (9 полей) │  промпт: classifier_pass1.txt (~4.4K, русский)
                    └────┬─────┘
                         │
              type in {fact,note,link}?
                    ┌────▼─────┐
                    │  Pass 2a  │  sub_type (одно поле)
                    │ per-type  │  промпт: pass2_sub_{type}.txt (~600-900 символов)
                    └──────────┘
                         │
              entity_hint or known entity?
                    ┌────▼─────┐
                    │  Pass 2b  │  entity names (плоский массив строк)
                    │ entities  │  промпт: pass2_entities.txt (английский)
                    │           │  format: JSON schema array of strings
                    └──────────┘
                         │
                    code-level matching: alias lookup → canonical names + types
```

**Поток обработки:**
1. Пользователь отправляет сообщение в Telegram
2. `bot.py` логирует сырой ввод в `raw_log`
3. `classifier.py` Pass 1: intent + type + content + url + remind_at + entity_hint
4. Pass 2a (если type ∈ {fact,note,link}): sub_type по per-type промпту
5. Pass 2b (если entity_hint или known entity): извлечение имён, code-level matching по aliases
6. `bot.py` корректирует относительное время через dateparser
7. `db.py` сохраняет запись в `memories` с привязкой сущностей
8. `scheduler.py` раз в минуту проверяет наступившие напоминания

## Типы записей

| Type | Описание | Sub_types (Pass 2a) |
|---|---|---|
| `reminder` | Напоминание с временем | — |
| `link` | Веб-ссылка | read_later, manual, reference, other |
| `fact` | Конкретная информация (параметры, действия, проекты) | params, credential, action, project, other |
| `note` | Заметка для отложенного просмотра | todo, info, other |
| `secret` | Пароль, токен (downgrade to fact/credential) | — |

## Структура проекта

```
ast-cj/
├── app/
│   ├── bot.py                      # основной обработчик, команды
│   ├── classifier.py               # 3-pass LLM-классификатор
│   ├── db.py                       # слой БД, CRUD, миграции
│   ├── scheduler.py                # планировщик напоминаний
│   └── prompts/
│       ├── classifier_pass1.txt    # промпт Pass 1 (intent + type)
│       ├── pass2_sub_fact.txt      # промпт Pass 2a для fact
│       ├── pass2_sub_note.txt      # промпт Pass 2a для note
│       ├── pass2_sub_link.txt      # промпт Pass 2a для link
│       ├── pass2_entities.txt      # промпт Pass 2b (entity extraction, английский)
│       └── classifier_v2.txt        # legacy промпт (fallback)
├── tests/
│   ├── batch_test.py               # batch-тестирование (3-pass, --pass {1,2a,2b,all})
│   ├── test_cases.json             # 39 тест-кейсов
│   └── pyproject.toml              # конфиг для импортов
├── docs/
│   ├── summary_1_project_status.md
│   └── summary_2_task_split.md
├── .env.example                    # пример конфигурации
└── README.md
```

## Установка и запуск

### Требования
- Python 3.12+
- [Ollama](https://ollama.ai) с загруженной моделью `qwen3.5:4b-q4_K_M`
- Telegram Bot Token (через [@BotFather](https://t.me/BotFather))

### Конфигурация

Скопируйте `.env.example` в `.env` и заполните:

```bash
TELEGRAM_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=qwen3.5:4b-q4_K_M
DB_PATH=/path/to/memories.db
OWNER_ID=123456789
# Если Telegram API недоступен напрямую (прокси):
PROXY_URL=socks5://100.x.x.x:1080
```

### Установка зависимостей

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install python-telegram-bot==21.5 httpx==0.27.0 APScheduler==3.10.4 dateparser python-dotenv
```

### Запуск

```bash
python3 bot.py
```

Первый запрос может занять 10–30 секунд (холодный старт модели). Последующие — 2–5 секунд.

## Тестирование

```bash
cd tests
python3 batch_test.py test_cases.json --model qwen3.5:4b-q4_K_M
```

Опции:
- `--pass {1,2a,2b,all}` — запуск конкретного прохода
- `--model` — указать альтернативную модель (gemma3:4b, qwen2.5:7b)
- `--db` — путь к БД для Pass 2b (entity matching с реестром)
- `--verbose` — показать raw output модели

Результаты: `results_<model>_<YYMMDDhhmm>.json`, только FAIL: `failed_<model>_<YYMMDDhhmm>.json`

## Структура базы данных

| Таблица | Назначение |
|---|---|
| `memories` | Основные записи (type, sub_type, content, tags, remind_at, is_done, deleted_at) |
| `raw_log` | Лог всех входящих сообщений (raw_input, llm_json, memory_id, tg_message_id) |
| `entities` | Сущности (name, type) |
| `entity_aliases` | Алиасы сущностей |
| `memory_entities` | Связь записей и сущностей (many-to-many) |
| `conversation_turns` | Контекст диалога (логирование хода) |

Миграции выполняются автоматически при запуске (`init_db`).

## Ключевые технические решения

- **Трёхпроходная классификация** — каждый промпт фокусирован на одной задаче
- **Per-type промпты для sub_type** — только релевантные правила и примеры
- **Entity extraction как flat array** — максимально простая структура для модели
- **JSON schema для Pass 2b** — `format: {"type":"array","items":{"type":"string"}}` принудительно
- **_normalize_entity_output()** — fallback для всех наблюденных паттернов моделей
- **Реестр убран из промпта** — модель извлекает по смыслу, код матчит по aliases
- **Локальная LLM** — privacy-first, данные не покидают сервер
- **`think: False`** — обязательно для qwen3.5:4b
- **dateparser для времени** — модель не умеет считать относительные даты
- **Мягкое удаление** — `deleted_at`, возможность восстановления
- **Message split** — автосплит до 3 сообщений по 4000 символов
- **Secret handling** — пароли не хранятся, downgrade + mask

## Лицензия

Разрешается Использование в личных и образовательных целях. 
Обязательна ссылка на источник.
