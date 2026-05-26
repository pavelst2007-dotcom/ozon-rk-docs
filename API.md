# Ozon RK — Public API v1

Публичный REST API для сторонних интеграций. Через него внешний сервис может получать список клиентов и их отслеживаемые товары (название, offer_id, SKU, ссылка) для синхронизации, проверок цен и других задач.

> **Версия:** v1
> **Base URL:** `https://<ваш-домен>/api/v1`
> **Формат:** JSON (UTF-8), все ответы содержат поле `ok: true|false`

---

## 1. Авторизация

Все запросы должны содержать заголовок:

```
Authorization: Bearer <ваш-токен>
```

Токен — глобальный (один на всю систему). Его выдаёт администратор через админ-панель (`/admin` → раздел «API-доступ»).

Формат токена: `ozn_live_` + 48 hex-символов (всего 57 символов).
Пример: `ozn_live_3f1a8b…<48 hex>`.

**Важно:**
- Токен показывается администратору **только один раз** при перевыпуске. Сохраните его в безопасном хранилище секретов вашего сервиса.
- Если токен скомпрометирован — попросите администратора нажать «Перевыпустить токен». Старый токен моментально перестанет работать.

---

## 2. Идентификатор клиента (`client_id`)

Каждому клиенту присвоен `public_id` — стабильный UUID-like идентификатор (32 hex-символа), который используется во всех запросах. Этот ID **не меняется** при правках email/имени.

Получить `public_id` всех клиентов можно через `GET /clients` (см. ниже).

---

## 3. Лимит запросов (Rate limit)

- **60 запросов в минуту** на токен.
- Сверяйтесь с заголовками ответа:
  - `X-RateLimit-Limit: 60`
  - `X-RateLimit-Remaining: <осталось>`
  - `X-RateLimit-Reset: <unix timestamp когда счётчик обнулится>`
- При превышении сервер вернёт **HTTP 429** + заголовок `Retry-After: <секунд>`.

Рекомендация: используйте экспоненциальный backoff при получении 429 и кэшируйте часто запрашиваемые данные на стороне клиента.

---

## 4. Эндпоинты

### 4.1. `GET /clients` — список клиентов

Возвращает всех клиентов системы.

**Запрос:**
```bash
curl -H "Authorization: Bearer <ваш-токен>" \
  https://<ваш-домен>/api/v1/clients
```

**Ответ `200 OK`:**
```json
{
  "ok": true,
  "count": 2,
  "clients": [
    {
      "id": "8f3a1c9d2e4b4a8b9c0d1e2f3a4b5c6d",
      "email": "client1@example.com",
      "name": "ООО Ромашка",
      "plan": "basic",
      "max_tracked_products": 100,
      "is_active": true
    },
    {
      "id": "1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d",
      "email": "client2@example.com",
      "name": "",
      "plan": "demo",
      "max_tracked_products": 10,
      "is_active": true
    }
  ]
}
```

**Поля клиента:**
| Поле | Тип | Описание |
|---|---|---|
| `id` | string | `public_id` — используйте в дальнейших запросах |
| `email` | string | email клиента |
| `name` | string | имя/название клиента (может быть пустым) |
| `plan` | string | тариф: `demo`, `basic`, `pro`, `custom` |
| `max_tracked_products` | number | лимит отслеживаемых товаров |
| `is_active` | boolean | активен ли аккаунт |

---

### 4.2. `GET /clients/:client_id/tracked` — отслеживаемые товары клиента

Список всех товаров, которые конкретный клиент добавил в мониторинг цен.

**Запрос:**
```bash
curl -H "Authorization: Bearer <ваш-токен>" \
  https://<ваш-домен>/api/v1/clients/8f3a1c9d2e4b4a8b9c0d1e2f3a4b5c6d/tracked
```

**Ответ `200 OK`:**
```json
{
  "ok": true,
  "client": {
    "id": "8f3a1c9d2e4b4a8b9c0d1e2f3a4b5c6d",
    "email": "client1@example.com",
    "name": "ООО Ромашка",
    "plan": "basic",
    "max_tracked_products": 100,
    "is_active": true
  },
  "count": 2,
  "items": [
    {
      "product_id": 4567378860,
      "name": "Кружка керамическая 350мл",
      "offer_id": "MUG-350-WHITE",
      "sku": "1234567890",
      "product_url": "https://www.ozon.ru/product/kruzhka-keramicheskaya-4567378860/",
      "url_status": "ok",
      "last_scan_at": "2026-05-26T11:42:18.000Z",
      "added_at": "2026-05-20T09:15:03.000Z"
    },
    {
      "product_id": 1234567890,
      "name": "Чайник электрический 1.7л",
      "offer_id": "KETTLE-17",
      "sku": "9876543210",
      "product_url": "https://www.ozon.ru/product/chaynik-1234567890/",
      "url_status": "ok",
      "last_scan_at": "2026-05-26T11:42:25.000Z",
      "added_at": "2026-05-21T14:02:11.000Z"
    }
  ]
}
```

**Поля товара (`items[]`):**
| Поле | Тип | Описание |
|---|---|---|
| `product_id` | number | ID товара на Ozon |
| `name` | string | название товара |
| `offer_id` | string | артикул продавца |
| `sku` | string | SKU Ozon |
| `product_url` | string | прямая ссылка на товар |
| `url_status` | string | `ok` / `invalid` / `redirect` |
| `last_scan_at` | string (ISO 8601) | время последнего сканирования цены |
| `added_at` | string (ISO 8601) | когда товар добавлен в мониторинг |

---

### 4.3. `GET /clients/:client_id/tracked/:product_id` — детали одного товара

Информация о конкретном товаре конкретного клиента. Удобно для точечной синхронизации.

**Запрос:**
```bash
curl -H "Authorization: Bearer <ваш-токен>" \
  https://<ваш-домен>/api/v1/clients/8f3a1c9d2e4b4a8b9c0d1e2f3a4b5c6d/tracked/4567378860
```

**Ответ `200 OK`:**
```json
{
  "ok": true,
  "client": { "id": "8f3a…", "email": "client1@example.com", "name": "ООО Ромашка", "plan": "basic", "max_tracked_products": 100, "is_active": true },
  "item": {
    "product_id": 4567378860,
    "name": "Кружка керамическая 350мл",
    "offer_id": "MUG-350-WHITE",
    "sku": "1234567890",
    "product_url": "https://www.ozon.ru/product/kruzhka-keramicheskaya-4567378860/",
    "url_status": "ok",
    "last_scan_at": "2026-05-26T11:42:18.000Z",
    "added_at": "2026-05-20T09:15:03.000Z"
  }
}
```

---

## 5. Коды ошибок

Все ошибки возвращают JSON формата:
```json
{ "ok": false, "error": "<error_code>", "message": "<человекочитаемое описание>" }
```

| HTTP | `error` | Когда |
|---|---|---|
| `400` | `bad_request` | некорректные параметры (например, нет `public_id`) |
| `401` | `unauthorized` | отсутствует заголовок `Authorization` |
| `401` | `invalid_token` | токен передан, но не совпадает |
| `404` | `client_not_found` | клиент с таким `public_id` не существует |
| `404` | `product_not_found` | товар не найден или не отслеживается этим клиентом |
| `429` | `rate_limit_exceeded` | превышен лимит 60 req/min (см. `Retry-After`) |
| `503` | `no_token_configured` | администратор ещё не настроил глобальный токен |
| `500` | `internal` | внутренняя ошибка сервера |

---

## 6. Примеры на разных языках

### Node.js (fetch)
```js
const TOKEN = process.env.OZON_RK_TOKEN;
const BASE  = 'https://<ваш-домен>/api/v1';

async function getClients() {
  const r = await fetch(`${BASE}/clients`, {
    headers: { Authorization: `Bearer ${TOKEN}` },
  });
  if (!r.ok) throw new Error(`HTTP ${r.status}: ${await r.text()}`);
  return r.json();
}

async function getTrackedProducts(clientId) {
  const r = await fetch(`${BASE}/clients/${clientId}/tracked`, {
    headers: { Authorization: `Bearer ${TOKEN}` },
  });
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}
```

### Python (requests)
```python
import os, requests

TOKEN = os.environ['OZON_RK_TOKEN']
BASE  = 'https://<ваш-домен>/api/v1'
HEADERS = { 'Authorization': f'Bearer {TOKEN}' }

def get_clients():
    r = requests.get(f'{BASE}/clients', headers=HEADERS, timeout=10)
    r.raise_for_status()
    return r.json()

def get_tracked(client_id: str):
    r = requests.get(f'{BASE}/clients/{client_id}/tracked', headers=HEADERS, timeout=10)
    r.raise_for_status()
    return r.json()
```

### PHP
```php
<?php
$token = getenv('OZON_RK_TOKEN');
$base  = 'https://<ваш-домен>/api/v1';

$ch = curl_init("$base/clients");
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ["Authorization: Bearer $token"],
]);
$response = curl_exec($ch);
curl_close($ch);
$data = json_decode($response, true);
```

---

## 7. Рекомендации по интеграции

1. **Кэшируйте список клиентов** на 5–15 минут — он меняется редко.
2. **Синхронизируйте товары инкрементально**: запоминайте `added_at` и при следующем запросе вы увидите новые товары в начале списка (сортировка `ORDER BY added_at DESC`).
3. **Не делайте больше 1 запроса в секунду** в нормальной работе — оставайтесь сильно ниже лимита 60 req/min.
4. **Храните токен в переменных окружения**, никогда не коммитьте его в репозиторий.
5. **При HTTP 429** ждите время из `Retry-After` и повторяйте запрос.
6. **При HTTP 401** не пытайтесь повторить — токен либо отозван, либо перевыпущен. Свяжитесь с администратором.

---

## 8. Что планируется в следующих версиях

- POST/PUT/DELETE эндпоинты для управления отслеживанием со стороны интегратора
- Webhook-уведомления о новых товарах и изменениях цен
- Доступ к истории цен (`GET /clients/:id/tracked/:product_id/price-history`)
- Per-client токены (вместо глобального)
- OpenAPI 3.0 спецификация + Swagger UI

Если нужна функциональность из этого списка — напишите администратору.

---

## 9. Контакты и поддержка

По вопросам интеграции: **pavelst2007@yandex.ru**

Документация на сервере: `https://<ваш-домен>/api-docs`
