# ТЗ: Facebook Conversions API - Purchase

## Цель

Отправлять на сервер Meta одно событие `Purchase` после успешной покупки.

Событие отправляется только после подтверждения оплаты/создания заказа на backend, не по клику на кнопку оплаты.

## Endpoint

```http
POST https://graph.facebook.com/v<GRAPH_API_VERSION>/<PIXEL_ID>/events
Content-Type: application/json
Authorization: Bearer <ACCESS_TOKEN>
```

Альтернативно токен можно передать query-параметром:

```http
POST https://graph.facebook.com/v<GRAPH_API_VERSION>/<PIXEL_ID>/events?access_token=<ACCESS_TOKEN>
```

Где:

| Параметр | Описание |
|---|---|
| `<GRAPH_API_VERSION>` | Версия Graph API, закрепленная в проекте. На 2026-07-06 текущая версия: `v25.0` |
| `<PIXEL_ID>` | ID Meta Pixel / Dataset |
| `<ACCESS_TOKEN>` | Access token с правом отправки событий в Conversions API |

## Событие

| Поле | Значение |
|---|---|
| `event_name` | `Purchase` |
| `action_source` | `website` |
| `event_time` | Unix timestamp в секундах, UTC |
| `event_id` | Уникальный ID покупки, желательно `order_id` |

`event_id` должен совпадать с `eventID` браузерного Meta Pixel события `Purchase`, если событие также отправляется из браузера. Это нужно для дедупликации.

`event_time` не должен быть старше 7 дней на момент отправки.

## Payload

```json
{
  "data": [
    {
      "event_name": "Purchase",
      "event_time": 1719835200,
      "event_id": "order_123456",
      "action_source": "website",
      "event_source_url": "https://example.com/checkout/success",
      "user_data": {
        "em": ["<sha256_email>"],
        "ph": ["<sha256_phone>"],
        "external_id": ["<user_id>"],
        "client_ip_address": "203.0.113.10",
        "client_user_agent": "Mozilla/5.0 ...",
        "fbp": "fb.1.1719834000000.1234567890",
        "fbc": "fb.1.1719834000000.AbCdEfGhIjKl"
      },
      "custom_data": {
        "currency": "USD",
        "value": 14.99,
        "order_id": "order_123456",
        "content_type": "product",
        "content_ids": ["max_monthly"],
        "contents": [
          {
            "id": "max_monthly",
            "quantity": 1,
            "item_price": 14.99
          }
        ]
      }
    }
  ]
}
```

## Обязательные поля

| Поле | Тип | Описание |
|---|---|---|
| `data` | array | Массив событий. Для этого ТЗ отправляем 1 событие |
| `data[].event_name` | string | Всегда `Purchase` |
| `data[].event_time` | integer | Время фактической покупки, Unix timestamp seconds |
| `data[].action_source` | string | Всегда `website` |
| `data[].event_source_url` | string | URL страницы, где произошла покупка. Для website-событий обязателен |
| `data[].user_data` | object | Данные пользователя для матчинга |
| `data[].custom_data.currency` | string | ISO 4217, например `USD`, `EUR`, `GBP` |
| `data[].custom_data.value` | number | Сумма покупки |

## Рекомендуемые поля

| Поле | Тип | Описание |
|---|---|---|
| `data[].event_id` | string | ID для дедупликации. Использовать `order_id` |
| `user_data.em` | array<string> | Email, нормализованный и SHA-256 hashed |
| `user_data.ph` | array<string> | Телефон, нормализованный и SHA-256 hashed |
| `user_data.external_id` | array<string> | Internal user ID, не хэшировать. Использовать стабильный ID пользователя в исходном формате |
| `user_data.client_ip_address` | string | IP пользователя, не хэшировать |
| `user_data.client_user_agent` | string | User-Agent пользователя, не хэшировать |
| `user_data.fbp` | string | Значение cookie `_fbp`, если есть |
| `user_data.fbc` | string | Значение cookie `_fbc`, если есть |
| `custom_data.order_id` | string | ID заказа |
| `custom_data.content_ids` | array<string> | ID купленного продукта/плана |
| `custom_data.contents` | array<object> | Детализация покупки |

## Нормализация user_data

Перед SHA-256, если значение хэшируется:

| Поле | Правило |
|---|---|
| `em` | trim, lowercase |
| `ph` | только цифры, включая country code |

`external_id` не хэшировать и не нормализовывать под SHA-256. Передавать стабильный внутренний user ID в том же исходном формате, который используется в системе. Если `external_id` также передается через Meta Pixel или другие интеграции, использовать там тот же raw ID.

Не хэшировать:

- `client_ip_address`
- `client_user_agent`
- `fbp`
- `fbc`

Если значения нет, поле не отправлять. Не отправлять `null`, пустые строки и пустые массивы.

## Сбор fbc без инициализированного Meta Pixel

`fbc` можно передать в `user_data.fbc`, даже если Meta Pixel library не была инициализирована и cookie `_fbc` не была создана автоматически.

Best practice: если пользователь пришел на сайт по ссылке с параметром `fbclid`, сохранить click ID как first-party cookie или в server-side session на первом landing page. Затем использовать это значение при отправке `Purchase` в Conversions API.

Формат значения `fbc`:

```text
fb.1.<creation_time_ms>.<fbclid>
```

Где:

| Часть | Описание |
|---|---|
| `fb` | Фиксированный prefix |
| `1` | Subdomain index. Для обычного сайта использовать `1` |
| `<creation_time_ms>` | Unix timestamp в миллисекундах на момент сохранения `fbclid` |
| `<fbclid>` | Значение query-параметра `fbclid` из landing URL |

Пример:

```text
fb.1.1719834000000.AbCdEfGhIjKl
```

### Client-side fallback

Если pixel script может быть не загружен, но JavaScript на сайте работает, можно создать аналог `_fbc` самостоятельно при наличии `fbclid` (далее делать выбор: fbc = _fbc || _fbca):

```js
const params = new URLSearchParams(window.location.search);
const fbclid = params.get("fbclid");

if (fbclid) {
  const fbc = `fb.1.${Date.now()}.${fbclid}`;

  document.cookie = [
    `_fbca=${encodeURIComponent(fbc)}`,
    "Max-Age=7776000",
    "Path=/",
    "SameSite=Lax",
    "Secure"
  ].join("; ");
}
```

`Max-Age=7776000` - 90 дней.

### Server-side fallback

Если есть backend middleware на входящих запросах, предпочтительно дополнительно сохранять `fbclid`/`fbc` на backend:

1. На каждом request читать `fbclid` из query string.
2. Если `fbclid` есть, сформировать `fbc` в формате `fb.1.<timestamp_ms>.<fbclid>`.
3. Сохранить `fbc` в first-party cookie и/или server-side session, привязанную к anonymous/session/user ID.
4. При создании заказа записать актуальное `fbc` в order attribution snapshot.
5. При отправке `Purchase` в CAPI передать это значение в `user_data.fbc`.

### Best practices

- Не хэшировать `fbc`; отправлять строку как есть.
- Собирать `fbclid` на первом landing page до редиректов, canonical redirect, смены языка, payment redirect и очистки query-параметров.
- Если сайт SPA, обрабатывать `fbclid` при первом page load и при client-side navigation.
- Если пользователь переходит между поддоменами, cookie должна быть доступна на нужном домене, например через `Domain=.example.com`, если это допустимо для проекта.
- Если пришел новый `fbclid`, обновить `_fbc` новым значением; если нового `fbclid` нет, не перезаписывать существующую `_fbc`.
- На checkout/payment flow сохранять `fbc` в backend session или order attribution snapshot, потому что browser cookie может потеряться на внешнем платежном редиректе.
- Учитывать consent/CMP: сохранять и отправлять `fbc` только если это разрешено политикой consent для advertising/marketing cookies.
- Не отправлять пустой, обрезанный или URL-encoded `fbc` в CAPI. Перед отправкой декодировать cookie value и передавать строку формата `fb.1.<timestamp_ms>.<fbclid>`.

## Официальные server-side SDK Meta

Meta рекомендует использовать Meta Business SDK для серверной интеграции с Conversions API. SDK поддерживается самой Meta и доступен для:

- PHP: `facebook/php-business-sdk`
- Node.js: `facebook-nodejs-business-sdk`
- Java: `facebook-java-business-sdk`
- Python: `facebook_business`
- Ruby: `facebookbusiness`

Для PHP установка через Composer:

```bash
composer require facebook/php-business-sdk
```

При использовании Business SDK SDK сам выполняет hashing для customer information parameters, где это требуется Meta. `external_id` в этой интеграции передавать как raw ID без предварительного хэширования.

Минимальная версия PHP для CAPI-фич Business SDK: PHP `>= 7.2`. Поддержка PHP 5 в Business SDK deprecated.

## Пример curl

```bash
curl -X POST "https://graph.facebook.com/v<GRAPH_API_VERSION>/<PIXEL_ID>/events" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -d '{
    "data": [
      {
        "event_name": "Purchase",
        "event_time": 1719835200,
        "event_id": "order_123456",
        "action_source": "website",
        "event_source_url": "https://example.com/checkout/success",
        "user_data": {
          "em": ["<sha256_email>"],
          "client_ip_address": "203.0.113.10",
          "client_user_agent": "Mozilla/5.0 ...",
          "fbp": "fb.1.1719834000000.1234567890"
        },
        "custom_data": {
          "currency": "USD",
          "value": 14.99,
          "order_id": "order_123456",
          "content_type": "product",
          "content_ids": ["max_monthly"],
          "contents": [
            {
              "id": "max_monthly",
              "quantity": 1,
              "item_price": 14.99
            }
          ]
        }
      }
    ]
  }'
```

## Тестовая отправка

Для проверки в Events Manager можно добавить `test_event_code` на верхний уровень payload:

```json
{
  "data": [
    {
      "event_name": "Purchase"
    }
  ],
  "test_event_code": "<TEST_EVENT_CODE>"
}
```

В production `test_event_code` не отправлять.

## Ожидаемый ответ

Успешный ответ:

```json
{
  "events_received": 1,
  "messages": [],
  "fbtrace_id": "<trace_id>"
}
```

Если API вернул ошибку, событие считать неотправленным и логировать:

- HTTP status
- response body
- `fbtrace_id`, если есть
- `event_id`
- `order_id`

## Требования к реализации

1. Отправлять `Purchase` только один раз на один `order_id`.
2. Использовать backend-to-backend запрос. Access token не должен попадать во frontend.
3. Таймаут запроса: 3-5 секунд.
4. Retry делать только для сетевых ошибок и 5xx, не более 3 попыток.
5. Для 4xx retry не делать, ошибку логировать.
6. Не блокировать успешную оплату пользователя из-за ошибки отправки в Meta.
7. Хранить факт отправки/ошибки по `order_id` для аудита и защиты от дублей.

## Acceptance criteria

- После успешной оплаты backend отправляет `POST /<PIXEL_ID>/events`.
- В payload есть ровно одно событие `Purchase`.
- `event_time` передается в секундах, UTC.
- `event_id` равен `order_id` или другому стабильному ID покупки.
- `currency` и `value` соответствуют фактической оплате.
- `user_data` содержит минимум один идентификатор пользователя, лучше несколько: `em`, `external_id`, `fbp`, `fbc`, IP, User-Agent.
- Access token не виден в браузере и не попадает в клиентские логи.

## Ссылки

- Meta Conversions API, server event parameters: https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/server-event
- Meta Conversions API, customer information parameters: https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/customer-information-parameters
- Meta Conversions API, payload helper: https://developers.facebook.com/docs/marketing-api/conversions-api/payload-helper
- Meta Conversions API, using the API and Business SDK features: https://developers.facebook.com/docs/marketing-api/conversions-api/using-the-api
- Meta Business SDK, getting started: https://developers.facebook.com/docs/business-sdk/getting-started
- Meta PHP Business SDK: https://github.com/facebook/facebook-php-business-sdk
