# Настройка CORS для A³ Attache

Этот документ описывает настройку CORS для двух компонентов:
1. **YouTrack REST API** — для работы плагина в iframe
2. **S3/MinIO Storage** — для загрузки и скачивания файлов из браузера

---

# Часть 1: Настройка CORS для YouTrack REST API

## Проблема

YouTrack плагины работают в iframe, который может загружаться через `srcdoc` или специальный URL. В этом случае браузер отправляет `Origin: null` вместо правильного домена (например, `youtrack.office.lan`).

Это стандартное поведение браузера для:
- iframe с атрибутом `srcdoc`
- iframe с `data:` URL
- iframe, загруженные через `file://`
- Некоторые случаи sandboxed iframe

## Решение 1: Настройка CORS в YouTrack (Рекомендуется)

### Через REST API

Добавьте **два** origins в список разрешенных:
1. **Основной домен YouTrack** (например, `https://youtrack.office.lan`) - для обычных запросов из браузера
2. **`null`** - для запросов из iframe плагинов

```bash
# 1. Получить текущие настройки CORS
curl -X GET "https://youtrack.office.lan/api/admin/globalSettings?fields=restSettings(allowAllOrigins,allowedOrigins)" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Accept: application/json"

# 2. Обновить настройки CORS (добавить оба адреса)
curl -X POST "https://youtrack.office.lan/api/admin/globalSettings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "restSettings": {
      "allowedOrigins": ["https://youtrack.office.lan", "null"],
      "allowAllOrigins": false
    }
  }'
```

**Важно**: 
- Замените `https://youtrack.office.lan` на ваш реальный домен YouTrack
- Если у вас уже есть другие origins в списке, добавьте их тоже, иначе они будут удалены
- Оба значения (`https://youtrack.office.lan` и `null`) необходимы для корректной работы

### Через YouTrack Admin UI (Рекомендуется)

1. Перейдите в **Administration** → **Global Settings** → **REST API**
2. В разделе **CORS (Cross-Origin Resource Sharing)** / **Resource Sharing**:
   - Снимите галочку **Allow requests from all origins** (если установлена)
   - В поле **Allowed Origins** добавьте:
     - `https://youtrack.office.lan` (ваш основной домен)
     - `null` (для iframe плагинов)
3. Сохраните изменения

📖 **Официальная документация**: [YouTrack Server Configuration - Resource Sharing Settings](https://www.jetbrains.com/help/youtrack/server/2025.3/server-configuration-settings.html#resource-sharing-settings)

## Решение 2: Использование allowAllOrigins (Не рекомендуется для production)

Если вы в тестовой среде, можно временно разрешить все origins:

```bash
curl -X POST "https://youtrack.office.lan/api/admin/globalSettings/restSettings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "allowAllOrigins": true
  }'
```

⚠️ **Внимание**: Это небезопасно для production окружения!

## Решение 3: Настройка на уровне приложения (Альтернатива)

Если у вас есть доступ к конфигурации YouTrack сервера, можно настроить CORS на уровне веб-сервера (nginx/Apache).

### Пример для nginx:

```nginx
location /api/ {
    # Разрешить запросы от null origin (для iframe плагинов)
    if ($http_origin = "null") {
        add_header 'Access-Control-Allow-Origin' 'null' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
    }
    
    # Разрешить запросы от основного домена
    if ($http_origin ~* "^https://youtrack\.office\.lan$") {
        add_header 'Access-Control-Allow-Origin' "$http_origin" always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
    }
    
    # Обработка preflight запросов
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    
    proxy_pass http://youtrack-backend;
}
```

## Проверка настроек

После настройки CORS проверьте, что запросы работают:

```bash
# Проверка preflight запроса
curl -X OPTIONS "https://youtrack.office.lan/api/admin/projects" \
  -H "Origin: null" \
  -H "Access-Control-Request-Method: GET" \
  -v

# Проверка обычного запроса
curl -X GET "https://youtrack.office.lan/api/admin/projects" \
  -H "Origin: null" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -v
```

В ответе должны быть заголовки:
- `Access-Control-Allow-Origin: null`
- `Access-Control-Allow-Methods: GET, POST, ...`
- `Access-Control-Allow-Headers: ...`

## Безопасность

### Почему `null` origin безопасен в данном случае:

1. **Ограниченный контекст**: `null` origin появляется только для iframe, которые уже загружены в контексте YouTrack
2. **Аутентификация**: Все запросы к YouTrack API требуют токен авторизации
3. **Изоляция**: Плагины работают в изолированном контексте YouTrack

### Рекомендации:

- ✅ Используйте `allowedOrigins: ["https://youtrack.office.lan", "null"]` вместо `allowAllOrigins: true`
- ✅ Всегда требуйте аутентификацию для API запросов
- ✅ Регулярно проверяйте логи на подозрительную активность
- ❌ Не используйте `allowAllOrigins: true` в production

## Альтернативные решения (если CORS нельзя настроить)

Если по каким-то причинам нельзя изменить CORS настройки YouTrack:

1. **Proxy через backend плагина**: Все запросы к YouTrack API делать через backend часть плагина (HTTP handlers)
2. **Использовать host.fetchYouTrack**: Этот метод уже обходит CORS, так как запросы делаются через YouTrack, а не напрямую из iframe

## Дополнительная информация

- [YouTrack REST API CORS Documentation](https://www.jetbrains.com/help/youtrack/devportal/operations-api-admin-globalSettings-restSettings.html)
- [MDN: Origin header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin)
- [CORS and null origin](https://stackoverflow.com/questions/8456538/origin-null-is-not-allowed-by-access-control-allow-origin)

---

# Часть 2: Настройка CORS для S3/MinIO Storage

## Зачем нужно CORS для S3/MinIO

A³ Attache загружает и скачивает файлы напрямую из браузера используя presigned URLs. Для этого необходимо настроить CORS на стороне S3/MinIO, чтобы разрешить браузеру делать запросы к вашему хранилищу.

## AWS S3: Настройка CORS

### Через AWS Console

1. Откройте [AWS S3 Console](https://console.aws.amazon.com/s3/)
2. Выберите ваш bucket
3. Перейдите на вкладку **"Permissions"**
4. Прокрутите до секции **"Cross-origin resource sharing (CORS)"**
5. Нажмите **"Edit"**
6. Вставьте следующую конфигурацию:

```json
[
    {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
        "AllowedOrigins": ["https://youtrack.yourcompany.com"],
        "ExposeHeaders": [
            "Date",
            "ETag",
            "Content-Length",
            "Last-Modified",
            "x-amz-server-side-encryption",
            "x-amz-request-id",
            "x-amz-id-2"
        ],
        "MaxAgeSeconds": 3000
    }
]
```

**Важно:**
- Замените `https://youtrack.yourcompany.com` на URL вашего YouTrack сервера
- Заголовок `Date` в `ExposeHeaders` необходим для автоматической коррекции clock skew
- Для тестирования можно использовать `"*"` в `AllowedOrigins`, но в production указывайте конкретный домен

### Через AWS CLI

```bash
# Создайте файл cors.json
cat > cors.json << 'EOF'
{
    "CORSRules": [
        {
            "AllowedHeaders": ["*"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedOrigins": ["https://youtrack.yourcompany.com"],
            "ExposeHeaders": [
                "Date",
                "ETag",
                "Content-Length",
                "Last-Modified"
            ],
            "MaxAgeSeconds": 3000
        }
    ]
}
EOF

# Примените конфигурацию
aws s3api put-bucket-cors \
    --bucket your-bucket-name \
    --cors-configuration file://cors.json
```

## MinIO: Настройка CORS

### Через MinIO Console

1. Откройте MinIO Console (обычно `http://localhost:9001` или ваш MinIO URL)
2. Перейдите в **"Buckets"** → выберите ваш bucket
3. Перейдите на вкладку **"Anonymous"**
4. Нажмите **"Edit"** в секции CORS
5. Добавьте правила CORS

### Через MinIO Client (mc)

**Установите MinIO Client:**
```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```

**Настройте alias:**
```bash
mc alias set myminio http://localhost:9000 minioadmin minioadmin123
```

**Создайте файл cors.json:**
```json
{
    "CORSRules": [
        {
            "AllowedOrigins": ["https://youtrack.yourcompany.com"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedHeaders": ["*"],
            "ExposeHeaders": ["Date", "ETag", "Content-Length", "Last-Modified"],
            "MaxAgeSeconds": 3000
        }
    ]
}
```

**Примените конфигурацию:**
```bash
mc anonymous set-json cors.json myminio/your-bucket-name
```

**Для тестирования (разрешить все origins):**
```json
{
    "CORSRules": [
        {
            "AllowedOrigins": ["*"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedHeaders": ["*"],
            "ExposeHeaders": ["Date", "ETag"],
            "MaxAgeSeconds": 3000
        }
    ]
}
```

## Проверка CORS конфигурации

### Проверка через curl

```bash
# Замените URL на ваш S3/MinIO endpoint
curl -X OPTIONS "https://your-bucket.s3.amazonaws.com/" \
  -H "Origin: https://youtrack.yourcompany.com" \
  -H "Access-Control-Request-Method: GET" \
  -v
```

В ответе должны быть заголовки:
- `Access-Control-Allow-Origin: https://youtrack.yourcompany.com`
- `Access-Control-Allow-Methods: GET, PUT, POST, DELETE, HEAD`
- `Access-Control-Expose-Headers: Date, ETag, ...`

### Проверка через браузер

1. Откройте DevTools (F12) в браузере
2. Перейдите на вкладку **Network**
3. Попробуйте загрузить файл через A³ Attache
4. Проверьте запросы к S3/MinIO — они должны выполняться без CORS ошибок

## Типичные проблемы

### Ошибка: "CORS policy: No 'Access-Control-Allow-Origin' header"

**Причина:** CORS не настроен или настроен неправильно

**Решение:**
1. Проверьте, что CORS конфигурация применена к bucket
2. Убедитесь, что `AllowedOrigins` содержит URL вашего YouTrack
3. Для MinIO проверьте, что используется правильный bucket name

### Ошибка: "CORS policy: Response to preflight request doesn't pass"

**Причина:** Отсутствуют необходимые методы или заголовки в CORS конфигурации

**Решение:**
1. Добавьте все необходимые методы: `GET`, `PUT`, `POST`, `DELETE`, `HEAD`
2. Добавьте `AllowedHeaders: ["*"]`
3. Убедитесь, что `OPTIONS` метод разрешен (обычно включается автоматически)

### Ошибка: Clock skew detection не работает

**Причина:** Заголовок `Date` не включен в `ExposeHeaders`

**Решение:**
Добавьте `"Date"` в список `ExposeHeaders` в CORS конфигурации

## Безопасность

### Рекомендации для production:

- ✅ Указывайте конкретный домен YouTrack в `AllowedOrigins` вместо `"*"`
- ✅ Ограничьте `AllowedMethods` только необходимыми методами
- ✅ Используйте HTTPS для YouTrack и S3/MinIO endpoints
- ✅ Регулярно проверяйте CORS конфигурацию
- ❌ Не используйте `"AllowedOrigins": ["*"]` в production

### Для тестирования:

Можно временно использовать `"*"` в `AllowedOrigins`, но обязательно замените на конкретный домен перед развертыванием в production.

## Дополнительная информация

- [AWS S3 CORS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html)
- [MinIO CORS Documentation](https://min.io/docs/minio/linux/administration/object-management.html#minio-bucket-cors)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
