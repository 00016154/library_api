# Library API spec document

### **Entities**

1. **Book**
    - id (unique identifier)
    - title (book title)
    - author_id (reference to the author)
    - category_id (reference to the category)
    - isbn
    - publication_year
    - available_copies (number of available copies)
2. **Author**
    - id
    - name
    - birth_date
    - bio (biography)
3. **Category**
    - id
    - name
    - description
4. **User** (for authentication and booking)
    - id
    - username
    - email
    - role (e.g., "user" or "admin")
5. **Rental**
    - id
    - book_id
    - user_id
    - rental_date
    - return_date
    - status (e.g., "active", "returned", "overdue")

---

### **Operations**

- CRUD operations for books, authors, and categories.
- Search books by various criteria (filtering).
- Book reservation/return.
- Get a list of available books.
- Get a list of books by author or category.

## DESIGN API

### **1. Ресурсы и URL-структура**

Придерживаемся принципов REST:

- **Коллекции** во множественном числе (**`/books`**).
- **Элементы** через ID (**`/books/{id}`**).
- **Вложенные ресурсы** (например, книги автора: **`/authors/{id}/books`**).

### **2. Эндпоинты и методы**

### **Книги (Books)**

| **Метод** | **Путь** | **Описание** | **Статус-коды** |
| --- | --- | --- | --- |
| GET | **`/books`** | Список книг (с фильтрами) | 200 OK |
| GET | **`/books/{id}`** | Получить книгу по ID | 200 OK, 404 Not Found |
| POST | **`/books`** | Добавить новую книгу | 201 Created, 400 Bad Request |
| PUT | **`/books/{id}`** | Обновить книгу (полностью) | 200 OK, 404 Not Found |
| PATCH | **`/books/{id}`** | Частично обновить книгу | 200 OK, 404 Not Found |
| DELETE | **`/books/{id}`** | Удалить книгу | 204 No Content, 404 Not Found |

### **Авторы (Authors)**

| **Метод** | **Путь** | **Описание** | **Статус-коды** |
| --- | --- | --- | --- |
| GET | **`/authors`** | Список авторов | 200 OK |
| GET | **`/authors/{id}/books`** | Книги конкретного автора | 200 OK, 404 Not Found |
| POST | **`/authors`** | Добавить автора | 201 Created |

### **Пользователи (Users)**

**Эндпоинты для управления аккаунтами и ролями:**

| **Метод** | **Путь** | **Описание** | **Статус-коды** |
| --- | --- | --- | --- |
| POST | **`/auth/register`** | Регистрация пользователя | 201 Created, 400 Bad Request |
| POST | **`/auth/login`** | Авторизация (получение JWT-токена) | 200 OK, 401 Unauthorized |
| GET | **`/users`** | Список пользователей (только для админов) | 200 OK, 403 Forbidden |
| GET | **`/users/{id}`** | Профиль пользователя | 200 OK, 404 Not Found |
| GET | **`/users/{id}/rentals`** | История аренд пользователя | 200 OK, 404 Not Found |
| PATCH | **`/users/{id}/role`** | Изменить роль (например, **`user`** → **`admin`**) | 200 OK, 403 Forbidden |
| DELETE | **`/users/{id}`** | Удалить пользователя | 204 No Content, 403 Forbidden |
| POST | **`/auth/verify-email`** | Отправить письмо с верификационным кодом | 200 OK, 429 (слишком много запросов) |
| POST | **`/auth/confirm-email`** | Подтвердить email через код/ссылку | 200 OK, 400 (неверный код) |

---

### **Аренда (Rentals)**

**Эндпоинты для бронирования и возврата книг:**

| **Метод** | **Путь** | **Описание** | **Статус-коды** |
| --- | --- | --- | --- |
| GET | **`/rentals`** | Список всех аренд (админы) | 200 OK, 403 Forbidden |
| GET | **`/rentals/{id}`** | Детали аренды | 200 OK, 404 Not Found |
| POST | **`/rentals`** | Забронировать книгу | 201 Created, 400 Bad Request |
| PATCH | **`/rentals/{id}/return`** | Вернуть книгу (изменить статус на **`returned`**) | 200 OK, 404 Not Found |
| GET | **`/rentals/overdue`** | Просроченные аренды | 200 OK, 403 Forbidden |

### **Категории (Categories)**

Аналогично авторам, но с дополнительным эндпоинтом:

- **`GET /categories/{id}/books`** – книги в категории.

### **Аренда (Rentals)**

| **Метод** | **Путь** | **Описание** | **Статус-коды** |
| --- | --- | --- | --- |
| POST | **`/rentals`** | Забронировать книгу | 201 Created, 400 (если нет доступных копий) |
| PATCH | **`/rentals/{id}/return`** | Вернуть книгу | 200 OK, 404 Not Found |

### **3. Фильтрация, сортировка, пагинация**

Для коллекций (**`/books`**, **`/authors`**) поддерживаем:

- **Фильтры** (через query-параметры):
    - **`?author_id=1`** – книги автора.
    - **`?category_id=2`** – книги категории.
    - **`?available=true`** – только доступные.
- **Сортировка**:
    - **`?sort=title`** или **`?sort=-publication_year`** (обратный порядок).
- **Пагинация**:
    - **`?page=2&limit=10`** – 10 книг на странице.

**Пример запроса**:

```
GET /books?category_id=5&available=true&sort=-publication_year&page=1&limit=5
```

**Ответ**:

```
{
  "data": [...],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 5
  }
}
```

**Фильтры для `/rentals`:**

- **`?user_id=10`** – аренды конкретного пользователя.
- **`?status=active`** – только активные аренды.
- **`?book_id=5`** – аренды конкретной книги.
- **При создании аренды (`POST /rentals`):**
    - Проверка, что **`available_copies`** книги > 0.
    - Автоматическое уменьшение **`available_copies`** на 1.
- **При возврате книги (`PATCH /rentals/{id}/return`):**
    - Увеличение **`available_copies`** книги на 1.

**Пример ошибки при бронировании:**

```
{
  "error": "Bad Request",
  "message": "No available copies for book ID 5",
  "status": 400
}
```

### **Cхема аутентификации**

- **JWT-токен** в заголовке **`Authorization: Bearer {token}`**.
- **Роли:**
    - **`user`** – может бронировать/возвращать книги, просматривать свои аренды.
    - **`admin`** – полный доступ ко всем эндпоинтам.

### **4. Статус-коды**

- **`200 OK`** – успешный GET/PUT/PATCH.
- **`201 Created`** – успешное создание (POST).
- **`204 No Content`** – успешное удаление (DELETE).
- **`400 Bad Request`** – неверные данные (например, нет автора при создании книги).
- **`401 Unauthorized`** – нужна аутентификация.
- **`403 Forbidden`** – нет прав (например, удаление книги неадмином).
- **`404 Not Found`** – ресурс не существует.
- **`429 Too Many Requests`** – превышен лимит запросов.

### **5. Примеры запросов**

### **Создание книги:**

```
POST /books
Headers: { "Authorization": "Bearer {token}", "Content-Type": "application/json" }
Body:
{
  "title": "1984",
  "author_id": 10,
  "category_id": 3,
  "isbn": "978-0451524935"
}
```

### **Получение книг с фильтрами:**

```
GET /books?available=true&category_id=3
```

### **Richardson Maturity Model**

Уровень 3 (HATEOAS):

- В ответы добавляем ссылки на связанные ресурсы.

**Пример**:

```
{
  "id": 1,
  "title": "1984",
  "_links": {
    "self": "/books/1",
    "author": "/authors/10",
    "category": "/categories/3"
  }
}
```

### **Логика работы**

### **Регистрация (`POST /auth/register`):**

1. Пользователь регистрируется → получает ответ **`201 Created`**.
2. **Сразу после регистрации**:
    - Система отправляет письмо с **верификационным кодом** (или ссылкой).
    - В БД сохраняется хэш кода и его срок действия (например, 24 часа).

### **Подтверждение email (`POST /auth/confirm-email`):**

```
{
  "email": "viva@example.com",
  "code": "ABCD123" // или токен из ссылки
}
```

**Успешный ответ**:

- Поле **`is_verified`** меняется на **`true`**.
- Пользователь получает JWT-токен для доступа к API.

### **Защита эндпоинтов**

- **Для неверифицированных пользователей**:
    - Запрещаем бронирование книг (**`POST /rentals`** → **403 Forbidden**).
    - Разрешаем только:
        - Повторную отправку кода верификации.
        - Подтверждение email.

**Пример ошибки**:

```
{
  "error": "Forbidden",
  "message": "Email not verified. Check your inbox or request a new verification code.",
  "status": 403
}
```

### **Дополнительные механизмы**

### **Повторная отправка кода:**

```
POST /auth/verify-email
Body: { "email": "mashas12@example.com" }
```

### **Интеграция с текущими эндпоинтами**

- **Логин (`POST /auth/login`)**:CopyDownload
    - Если email не подтвержден → **403 Forbidden + сообщение**.
    
    ```
    {
      "error": "Forbidden",
      "message": "Email not verified. A new code has been sent to your email.",
      "status": 403
    }
    ```
    
- **Получение профиля (`GET /users/{id}`)**:
    - Поле **`is_verified`** отображается только для владельца профиля или админа.

### **Пример письма**

**Тема**: "Подтвердите ваш email для LibraryAPI"

**Тело письма**:

```
Здравствуйте, Masha!
Для завершения регистрации введите код: ABCD123
Или перейдите по ссылке: https://api.library/confirm-email?token=xyz123
```

## **Кэширование в REST API**

### **1. Стратегия кэширования**

Используем **HTTP-кэширование** (клиентское и серверное) + **Redis** для часто запрашиваемых данных.

### **Какие данные кэшируем?**

| **Ресурс** | **Метод** | **Условия кэширования** | **Время жизни (TTL)** |
| --- | --- | --- | --- |
| **`GET /books`** | GET | Фильтры, пагинация | 5 минут |
| **`GET /books/{id}`** | GET | Для неизменяемых книг (старые издания) | 1 час |
| **`GET /authors`** | GET | Без частых обновлений | 10 минут |
| **`GET /categories`** | GET | Статичные данные | 1 день |

---

### **2. HTTP-заголовки для кэширования**

Добавляем в ответы сервера:

- **`Cache-Control`** – управление кэшем на клиенте и прокси.
- **`ETag`** – хэш ресурса для проверки изменений.
- **`Last-Modified`** – дата последнего обновления.

**Пример для `GET /books`:**

```
HTTP/1.1 200 OKCache-Control: public, max-age=300ETag: "a1b2c3d4"Last-Modified: Wed, 21 Oct 2024 10:00:00 GMT
```

---

### **3. Реализация на стороне сервера**

### **Используем Redis для:**

- Кэширования списков книг/авторов с фильтрами.
- Хранения ETag для каждого ресурса.

**Псевдокод для `GET /books`:**

```
def get_books():
    cache_key = f"books:{filters}:{page}"
    cached_data = redis.get(cache_key)

    if cached_data:
        return jsonify(cached_data)

    # Если нет в кэше – запрос к БД
    books = db.query_books(filters)
    redis.setex(cache_key, ttl=300, value=books)
    return jsonify(books)
```

---

### **4. Инвалидация кэша**

Кэш должен обновляться при изменениях данных:

- **Для `POST/PUT/DELETE /books`**:CopyDownload
    
    Удаляем ключи Redis, связанные с книгами:
    
    python
    
    ```
    redis.delete("books:*")  # Инвалидируем все кэши списков
    redis.delete(f"book:{id}")  # Инвалидируем конкретную книгу
    ```
    

---

### **5. Условные запросы (Conditional GET)**

Клиент может проверить, изменились ли данные, используя **`ETag`** или **`Last-Modified`**:

```
GET /books/123
If-None-Match: "a1b2c3d4"
```

**Если ресурс не изменился**, сервер вернет:

```
HTTP/1.1 304 Not Modified
```

---

### **6. Примеры для разных эндпоинтов**

### **Для статичных данных (`GET /categories`):**

```
Cache-Control: public, max-age=86400  # 1 день
```

### **Для часто обновляемых (`GET /rentals/overdue`):**

```
Cache-Control: no-cache  # Сервер всегда проверяет актуальность
```

---

### **7. Кэширование ошибок**

- **404 Not Found** для несуществующих книг: CopyDownload
    
    Кэшируем на 1 минуту, чтобы избежать DDoS.
    
    ```
    HTTP/1.1 404 Not FoundCache-Control: public, max-age=60
    ```
