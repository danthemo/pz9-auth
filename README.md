# Практическое занятие № 9: Реализация регистрации и входа пользователей с bcrypt
Ходыч Даниил Евгеньевич ЭФМО-02-25

---

## Описание проекта

Реализован сервис аутентификации на Go для управления учётными записями пользователей. Сервис предоставляет:

- **Регистрация пользователей** с безопасным хешированием пароля через bcrypt
- **Вход пользователя** с проверкой учётных данных
- **Валидация email и пароля**
- **Обработка ошибок** с обобщёнными сообщениями для защиты от перебора email-адресов
- **Хранение паролей** только в виде bcrypt-хеша

---

## Структура проекта

```
.
├── cmd
│   └── api
│       └── main.go
├── docker-compose.yml
├── go.mod
├── go.sum
└── internal
    ├── core
    │   └── user.go
    ├── http
    │   └── handlers
    │       └── auth.go
    ├── platform
    │   └── config
    │       └── config.go
    └── repo
        ├── postgres.go
        └── user_repo.go
```

---

## Установка и запуск

### 1. Предварительные требования

- Go 1.21+
- Docker

### 2. Клонирование и подготовка

```bash
git clone https://github.com/danthemo/pz9-auth.git
cd pz9-auth
```

### 3. Запуск PostgreSQL через Docker

```bash
docker compose up -d
```

Проверка подключения:

```bash
docker exec -it pz9-postgres psql -U user -d pz9
# В psql:
\dt
\q
```

**Содержимое docker-compose.yml:**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: pz9-postgres
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: pz9
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 4. Переменные окружения

**Содержимое .env:**

```
DB_DSN=postgres://user:password@localhost:5432/pz9?sslmode=disable
BCRYPT_COST=12
APP_ADDR=:8082
```

### 5. Установка зависимостей

```bash
go mod tidy
```

### 6. Запуск сервера

```bash
go run ./cmd/api/main.go
```

---

## Примеры запросов

### 1. Регистрация пользователя (успешно)

**curl:**

```bash
curl -i -X POST http://5.129.194.73:8082/auth/register -H "Content-Type: application/json" -d "{\"email\":\"user@example.com\",\"password\":\"Secret123!\"}"
```

<img width="705" height="659" alt="image" src="https://github.com/user-attachments/assets/536dc23a-86dc-4e09-8b5c-71264d277814" />

---

### 2. Повторная регистрация (email уже занят)

**curl:**

```bash
curl -i -X POST http://5.129.194.73:8082/auth/register -H "Content-Type: application/json" -d "{\"email\":\"user@example.com\",\"password\":\"AnotherPass\"}"
```

<img width="637" height="593" alt="image" src="https://github.com/user-attachments/assets/ebd2024b-67ba-4cd1-8d56-9cbe92edd9ff" />

---

### 3. Вход с верными учётными данными

**curl:**

```bash
curl -i -X POST http://5.129.194.73:8082/auth/login -H "Content-Type: application/json" -d "{\"email\":\"user@example.com\",\"password\":\"Secret123!\"}"
```

<img width="628" height="675" alt="image" src="https://github.com/user-attachments/assets/9e064b16-8ed0-4270-b782-545d7eb5625d" />

---

### 4. Вход с неверным паролем

**curl:**

```bash
curl -i -X POST http://5.129.194.73:8082/auth/login -H "Content-Type: application/json" -d "{\"email\":\"user@example.com\",\"password\":\"WrongPassword\"}"
```

<img width="620" height="624" alt="image" src="https://github.com/user-attachments/assets/c9f1d42d-d891-4920-bc45-f26c2e8ec710" />

---

## Код: Ключевые компоненты

### 1. Модель пользователя (internal/core/user.go)

```go
package core

import "time"

type User struct {
    ID           int64     `gorm:"primaryKey" json:"id"`
    Email        string    `gorm:"uniqueIndex;size:255;not null" json:"email"`
    PasswordHash string    `gorm:"size:255;not null" json:"-"`
    CreatedAt    time.Time `json:"createdAt"`
    UpdatedAt    time.Time `json:"updatedAt"`
}
```

- `json:"-"` означает, что поле `PasswordHash` никогда не выдаётся в HTTP ответах.

---

### 2. Обработчик регистрации (internal/http/handlers/auth.go)

```go
type registerReq struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type authResp struct {
    Status string      `json:"status"`
    User   interface{} `json:"user,omitempty"`
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var in registerReq
    if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
        writeErr(w, http.StatusBadRequest, "invalid_json")
        return
    }

    in.Email = strings.TrimSpace(strings.ToLower(in.Email))
    if in.Email == "" || len(in.Password) < 8 {
        writeErr(w, http.StatusBadRequest, "email_required_and_password_min_8")
        return
    }

    hash, err := bcrypt.GenerateFromPassword([]byte(in.Password), h.BcryptCost)
    if err != nil {
        writeErr(w, http.StatusInternalServerError, "hash_failed")
        return
    }

    u := core.User{Email: in.Email, PasswordHash: string(hash)}
    if err := h.Users.Create(r.Context(), &u); err != nil {
        if err == repo.ErrEmailTaken {
            writeErr(w, http.StatusConflict, "email_taken")
            return
        }
        writeErr(w, http.StatusInternalServerError, "db_error")
        return
    }

    writeJSON(w, http.StatusCreated, authResp{
        Status: "ok",
        User: map[string]any{"id": u.ID, "email": u.Email},
    })
}
```

- `bcrypt.GenerateFromPassword()` принимает пароль и cost, возвращает хеш
- В БД сохраняем **только хеш**
- Если email уже занят -> возвращаем 409

---

### 3. Обработчик входа (internal/http/handlers/auth.go)

```go
type loginReq struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var in loginReq
    if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
        writeErr(w, http.StatusBadRequest, "invalid_json")
        return
    }

    in.Email = strings.TrimSpace(strings.ToLower(in.Email))
    if in.Email == "" || in.Password == "" {
        writeErr(w, http.StatusBadRequest, "email_and_password_required")
        return
    }

    u, err := h.Users.ByEmail(context.Background(), in.Email)
    if err != nil {
        writeErr(w, http.StatusUnauthorized, "invalid_credentials")
        return
    }

    if bcrypt.CompareHashAndPassword([]byte(u.PasswordHash), []byte(in.Password)) != nil {
        writeErr(w, http.StatusUnauthorized, "invalid_credentials")
        return
    }

    writeJSON(w, http.StatusOK, authResp{
        Status: "ok",
        User: map[string]any{"id": u.ID, "email": u.Email},
    })
}
```

- `bcrypt.CompareHashAndPassword()` сравнивает хеш и пароль
- Если пароль неверный или email не найден -> один ответ: "invalid_credentials"
- Это защищает от перебора email-адресов

---

### 4. Repository для работы с БД (internal/repo/user_repo.go)

```go
package repo

import (
    "context"
    "errors"
    "gorm.io/gorm"
    "example.com/pz9-auth/internal/core"
)

var ErrUserNotFound = errors.New("user not found")
var ErrEmailTaken = errors.New("email already in use")

type UserRepo struct{ db *gorm.DB }

func NewUserRepo(db *gorm.DB) *UserRepo { 
    return &UserRepo{db: db} 
}

func (r *UserRepo) AutoMigrate() error {
    return r.db.AutoMigrate(&core.User{})
}

func (r *UserRepo) Create(ctx context.Context, u *core.User) error {
    if err := r.db.WithContext(ctx).Create(u).Error; err != nil {
        if errors.Is(err, gorm.ErrDuplicatedKey) {
            return ErrEmailTaken
        }
        return err
    }
    return nil
}

func (r *UserRepo) ByEmail(ctx context.Context, email string) (core.User, error) {
    var u core.User
    err := r.db.WithContext(ctx).Where("email = ?", email).First(&u).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return core.User{}, ErrUserNotFound
    }
    return u, err
}
```

---

### 5. Запуск сервера (cmd/api/main.go)

```go
package main

import (
    "log"
    "net/http"
    "github.com/go-chi/chi/v5"
    "example.com/pz9-auth/internal/platform/config"
    "example.com/pz9-auth/internal/http/handlers"
    "example.com/pz9-auth/internal/repo"
)

func main() {
    cfg := config.Load()

    db, err := repo.Open(cfg.DB_DSN)
    if err != nil { 
        log.Fatal("db connect:", err) 
    }

    users := repo.NewUserRepo(db)
    if err := users.AutoMigrate(); err != nil { 
        log.Fatal("migrate:", err) 
    }

    auth := &handlers.AuthHandler{Users: users, BcryptCost: cfg.BcryptCost}

    r := chi.NewRouter()
    r.Post("/auth/register", auth.Register)
    r.Post("/auth/login", auth.Login)

    log.Println("listening on", cfg.Addr)
    log.Fatal(http.ListenAndServe(cfg.Addr, r))
}
```

---

## SQL и Миграции

**Создаваемая таблица (через GORM AutoMigrate):**

```bash
docker exec -it pz9-postgres psql -U user -d pz9
pz9=# \dt
pz9=# SELECT * FROM users;
```

<img width="570" height="297" alt="image" src="https://github.com/user-attachments/assets/c6b7f934-727b-4d48-9b9a-7ff8310d25dd" />

---

## Тестирование

### Автоматические тесты

```bash
go test ./internal/repo
```

---

## Выводы и ответы на контрольные вопросы

### 1. В чём разница между хранением пароля и хранением его хеша? Зачем соль? Почему bcrypt, а не SHA-256?

**Хранение пароля в открытом виде:**
- Если БД украдена -> все пароли видны
- Можно увидеть пароль в логах, в интерфейсе админа и т.д.

**Хеширование:**
- Одностороннее преобразование: из хеша невозможно вернуть пароль
- Даже админ БД не знает пароли пользователей

**Зачем соль:**
- Если два пользователя выберут пароль "123456", их хеши должны быть разными

**Почему bcrypt, а не SHA-256:**
- SHA-256 очень быстрый -> легко подобрать пароль brute force
- bcrypt специально медленный -> затратно перебирать
- bcrypt имеет встроенную соль и регулируемую сложность

---

### 2. Что произойдёт при снижении/повышении cost у bcrypt? Как подобрать значение?

**Снижение cost (например, с 12 на 10):**
- Хеширование ускорится
- Быстрее регистрация и вход для пользователя
- **Но:** хакер может быстрее перебирать пароли

**Повышение cost (например, с 12 на 14):**
- Хеширование замедлится
- Медленнее регистрация и вход для пользователя
- **Но:** хакер не сможет быстро перебирать пароли

---

### 3. Какие статусы и ответы должны возвращать POST /auth/register и POST /auth/login в типичных сценариях?

**POST /auth/register:**

| Сценарий | Статус | Ответ |
|----------|--------|-------|
| Успешная регистрация | 201 Created | `{"status":"ok","user":{"id":1,"email":"user@example.com"}}` |
| Email уже занят | 409 Conflict | `{"error":"email_taken"}` |
| Невалидные данные | 400 Bad Request | `{"error":"email_required_and_password_min_8"}` |
| Ошибка БД | 500 Internal Server Error | `{"error":"db_error"}` |

**POST /auth/login:**

| Сценарий | Статус | Ответ |
|----------|--------|-------|
| Вход успешен | 200 OK | `{"status":"ok","user":{"id":1,"email":"user@example.com"}}` |
| Неверные учётные данные | 401 Unauthorized | `{"error":"invalid_credentials"}` |
| Невалидный JSON | 400 Bad Request | `{"error":"invalid_json"}` |

---

### 4. Какие риски несут подробные сообщения об ошибках при логине?

**Подробные сообщения** могут сообщить хакерам нужную информацию, например, для перебора email адресов, зарегестрированных на ресурсе

**Правильный подход:**
```json
{"error": "invalid_credentials"}
```
- Один ответ для всех случаев ошибки (email не найден или пароль неверный)
- Хакер не узнает, какие email-адреса в системе
