# LinkedIn publishing integration

## Текущий статус

В текущем MVP фактическая публикация в LinkedIn не встроена.

Кнопка **Публиковать** выполняет MVP-логику:

1. сохраняет финальный текст и выбранное изображение;
2. создает запись в `approved_posts`;
3. переводит результат в статус `approved / ready_to_publish`;
4. показывает пользователю сообщение, что пост подтвержден и готов к публикации.

Это сделано намеренно: публикационный слой можно добавить отдельно, не переделывая основную генерационную логику.

## Что уже готово для публикации

После утверждения поста в базе есть все необходимые данные:

```text
session_id
topic_id
post_id
image_id
final_text
final_image_path
status
created_at
```

Эти данные можно использовать как очередь публикаций.

## Вариант 1. Прямая интеграция с LinkedIn API

### Публикация в личный профиль

Для публикации в личный профиль нужен:

- LinkedIn Developer App;
- OAuth 2.0;
- авторизация пользователя через 3-legged OAuth;
- scope `w_member_social`;
- Posts API.

Общая логика:

```text
User authorizes LinkedIn App
→ application receives access token
→ n8n / backend creates post through LinkedIn Posts API
→ LinkedIn returns post URN
→ publication status is saved in PostgreSQL
```

### Публикация на страницу компании

Для публикации от имени LinkedIn Company Page обычно нужно:

- LinkedIn Developer App;
- LinkedIn Company Page;
- пользователь с правами администратора страницы;
- scope `w_organization_social`;
- доступ к Community Management / Posts API;
- app review / approval LinkedIn, если требуется.

Общая логика похожа на личный профиль, но author будет organization URN, а не member URN.

## Пост с изображением

Для поста с изображением нужен дополнительный шаг загрузки media asset.

Общая логика:

```text
1. Получить OAuth access token.
2. Инициализировать загрузку изображения через Images API.
3. Получить upload URL и image URN.
4. Загрузить файл изображения на upload URL.
5. Создать post через Posts API.
6. Прикрепить image URN к посту.
7. Сохранить linkedin_post_urn и статус публикации.
```

## Что нужно добавить в базу для прямой интеграции

Для production-версии можно добавить таблицу `linkedin_accounts`.

Пример полей:

```text
id
telegram_user_id
linkedin_member_urn
access_token_encrypted
refresh_token_encrypted
token_expires_at
status
created_at
updated_at
```

И таблицу `linkedin_publications`.

Пример полей:

```text
id
approved_post_id
linkedin_account_id
author_type
author_urn
linkedin_post_urn
linkedin_image_urn
publish_status
publish_error
published_at
created_at
updated_at
```

Примеры статусов:

```text
draft
ready
publishing
published
failed
reauth_required
```

## Вариант 2. Сторонний publishing API

Для MVP можно использовать внешний publishing-сервис, который уже поддерживает LinkedIn.

Примеры:

- Post for Me
- PublishQ
- Buffer
- Hootsuite / аналогичные social media platforms

В этом варианте n8n не работает напрямую с LinkedIn API. Он отправляет утвержденный текст и изображение в сторонний сервис, а сервис публикует пост в LinkedIn.

Общая схема:

```text
Telegram Bot
→ n8n workflow
→ approved post in PostgreSQL
→ third-party publishing API
→ LinkedIn
→ save publish status
```

## Почему сторонний publishing API может быть удобен для MVP

Плюсы:

- быстрее проверить реальную автопубликацию;
- не нужно сразу проходить весь LinkedIn app approval process;
- меньше работы с OAuth, токенами, загрузкой media и versioned API;
- проще встроить через обычный HTTP Request в n8n;
- можно использовать как временный publication layer.

Минусы:

- дополнительная подписка;
- зависимость от внешнего сервиса;
- нужно проверить лимиты, тарифы и условия;
- для некоторых сценариев может понадобиться подключение аккаунта внутри стороннего сервиса.

## Важный момент про изображения

Если публикация идет через сторонний сервис, выбранная картинка должна быть доступна сервису.

Локального пути вроде этого недостаточно:

```text
/files/linkedin_images/image.png
```

Нужен публичный URL или загрузка файла в сервис.

Варианты решения:

- отдавать изображения через публичный static endpoint;
- хранить изображения в S3 / Cloudflare R2 / Supabase Storage;
- загружать media через API стороннего сервиса;
- добавить временные signed URLs.

## Очередь публикаций

Для production-версии лучше не публиковать сразу из основной ветки Telegram-сценария.

Лучше использовать отдельную очередь:

```text
approved_posts
→ publication queue
→ publishing worker / n8n workflow
→ LinkedIn / publishing API
→ update publication status
```

Это даст:

- повторные попытки при ошибке;
- публикацию по расписанию;
- понятные статусы;
- возможность ручной модерации;
- поддержку нескольких аккаунтов и страниц.

## Ограничения и риски

Нужно учитывать:

- LinkedIn API может требовать approval;
- права на Company Page должны быть корректными;
- токены могут истекать или быть отозваны;
- API может возвращать 401, 403, 409, 429 и временные ошибки;
- нужно следить за версией LinkedIn API;
- публикация через сторонние сервисы зависит от их тарифов и SLA;
- LinkedIn-сбор источников нужно делать с учетом правил платформы.

## Рекомендуемый путь для MVP

Практичный вариант:

1. оставить текущую генерационную логику без изменений;
2. использовать `approved_posts` как очередь;
3. проверить сторонний publishing API на тестовом LinkedIn-аккаунте;
4. после успешного теста добавить отдельный publishing workflow;
5. позже, при необходимости, заменить сторонний сервис на прямую LinkedIn API-интеграцию.

Такой подход позволяет быстро показать реальную автопубликацию и при этом не завязывать всю архитектуру на один конкретный сервис.
