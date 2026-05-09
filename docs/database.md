# Модель данных PostgreSQL

## Назначение базы

PostgreSQL используется как основное хранилище состояния и контекста.

База нужна, чтобы бот не терял данные между шагами:

- на каком этапе находится пользователь;
- какую тему он выбрал;
- какой язык выбран;
- какие темы уже показывались;
- какой текст сейчас актуален;
- какие версии текста были созданы;
- какие картинки уже сгенерированы;
- какая картинка выбрана;
- какие правки пользователь вносил;
- какой результат утвержден.

## Основные таблицы

```text
bot_sessions
topic_batches
post_topics
generated_posts
generated_images
approved_posts
workflow_logs
linkedin_sources
linkedin_source_chunks
```

## bot_sessions

Главная таблица состояния пользователя.

Хранит текущий этап сценария и ссылки на выбранные сущности.

Основные поля:

```text
id
telegram_user_id
chat_id
state
current_topic_id
current_post_id
current_image_id
selected_language
topics_batch_no
images_batch_no
created_at
updated_at
```

Примеры состояний:

```text
idle
generating_topics
choosing_topic
choosing_language
generating_post
choosing_image
preview
revising_text
revising_image
approved
```

## topic_batches

Хранит партии сгенерированных тем.

Одна партия = один набор из 12 тем.

Основные поля:

```text
id
session_id
batch_no
source_mode
created_at
```

Пример:

```text
source_mode = mixed
```

Это означает, что партия содержит и LinkedIn-сигналы, и интернет-сигналы.

## post_topics

Хранит сгенерированные темы.

В одной партии создается 12 тем:

- 1–6 — LinkedIn-related topics;
- 7–12 — internet topics.

Основные поля:

```text
id
session_id
batch_id
batch_no
topic_no
title
explanation
relevance_now
source_type
source_company
source_url
source_signal
score
is_selected
created_at
```

Назначение:

- показывать пользователю список тем;
- хранить источник темы;
- поддерживать антидубли;
- понимать, какая тема была выбрана.

## generated_posts

Хранит тексты постов и их версии.

При правке текста старая версия не удаляется. Создается новая версия.

Основные поля:

```text
id
session_id
topic_id
language
version
post_text
feedback
created_at
```

Пример логики версий:

```text
v1 — первый сгенерированный пост
v2 — пост после первой правки
v3 — пост после второй правки
```

## generated_images

Хранит image prompts, batch изображений и выбранное изображение.

Основные поля:

```text
id
session_id
post_id
batch_no
image_no
prompt
image_url
file_path
feedback
is_selected
created_at
```

Назначение:

- хранить 3 варианта изображения;
- поддерживать повторную генерацию картинок;
- сохранять выбранное изображение;
- не терять предыдущие варианты.

## approved_posts

Хранит финально утвержденный результат.

Основные поля:

```text
id
session_id
topic_id
post_id
image_id
final_text
final_image_path
status
created_at
```

В MVP кнопка **Публиковать** сохраняет запись в эту таблицу.

Пример статуса:

```text
approved
ready_to_publish
```

## workflow_logs

Таблица для технических логов workflow.

Основные поля:

```text
id
session_id
step
input_json
output_json
error_message
created_at
```

Назначение:

- фиксировать ошибки;
- понимать, на каком шаге произошел сбой;
- сохранять вход и ответ внешнего сервиса;
- упрощать отладку.

## linkedin_sources

Справочник LinkedIn-релевантных источников.

Хранит список публичных страниц компаний, которые используются как curated source pool.

Основные поля:

```text
id
company_key
company_name
company_url
category
is_active
last_status_code
last_parsed_at
raw_text_len
clean_text_len
chunks_count
parse_error
created_at
updated_at
```

Назначение:

- хранить список источников;
- понимать, какие источники дают полезный текст;
- отключать пустые или неработающие источники;
- не удалять источник полностью при временной ошибке.

Примеры категорий:

```text
co-development
qa-localization
engine-tools
production-tools
publisher-studio
mobile-liveops
platform-liveops
art-production
monetization-platform
```

## linkedin_source_chunks

Хранит очищенные текстовые chunks из LinkedIn-релевантных источников.

Основные поля:

```text
id
source_id
company_key
company_name
category
chunk_no
chunk_text
chunk_hash
is_active
created_at
updated_at
```

Назначение:

- хранить подготовленный текстовый корпус;
- не парсить LinkedIn во время пользовательского сценария;
- выбирать chunks с ротацией;
- балансировать источники по компаниям и категориям;
- повышать разнообразие тем.

## Почему выбран PostgreSQL

PostgreSQL удобен для этого проекта, потому что:

- хранит состояние пользователя;
- поддерживает связи между сессиями, темами, постами и изображениями;
- позволяет хранить версии;
- позволяет строить очереди публикаций;
- подходит для будущего масштабирования.

## Что можно добавить позже

Для production-версии можно добавить:

- `linkedin_accounts` — подключенные LinkedIn-аккаунты;
- `linkedin_publications` — очередь и статусы публикаций;
- embeddings для semantic duplicate detection;
- отдельные таблицы для брендов и tone-of-voice профилей;
- расширенное логирование внешних API.
