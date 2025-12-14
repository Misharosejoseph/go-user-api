# go-user-api
# Go Backend User API – Ready-to-Run GitHub Repository

.
├── cmd/server/main.go
├── config/config.go
├── db/migrations/001_create_users.sql
├── db/sqlc/schema.sql
├── db/sqlc/users.sql
├── db/sqlc/sqlc.yaml
├── internal/handler/user_handler.go
├── internal/middleware/request_logger.go
├── internal/models/user.go
├── internal/repository/user_repository.go
├── internal/routes/routes.go
├── internal/service/age.go
├── internal/service/user_service.go
├── internal/logger/logger.go
├── go.mod
├── go.sum
└── README.md
```

---

## 1️⃣ `go.mod`

```go
module github.com/yourusername/user-api

go 1.22

require (
    github.com/gofiber/fiber/v2 v2.52.0
    github.com/go-playground/validator/v10 v10.20.0
    go.uber.org/zap v1.26.0
    github.com/jackc/pgx/v5 v5.5.4
)
```

---

## 2️⃣ Database (PostgreSQL)

### `db/migrations/001_create_users.sql`

```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    dob DATE NOT NULL
);
```

### `db/sqlc/schema.sql`

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    dob DATE NOT NULL
);
```

### `db/sqlc/users.sql`

```sql
-- name: CreateUser :one
INSERT INTO users (name, dob)
VALUES ($1, $2)
RETURNING id, name, dob;

-- name: GetUser :one
SELECT id, name, dob FROM users WHERE id = $1;

-- name: ListUsers :many
SELECT id, name, dob FROM users ORDER BY id;

-- name: UpdateUser :one
UPDATE users SET name = $2, dob = $3 WHERE id = $1
RETURNING id, name, dob;

-- name: DeleteUser :exec
DELETE FROM users WHERE id = $1;
```

### `db/sqlc/sqlc.yaml`

```yaml
version: "2"
sql:
  - engine: postgresql
    schema: "schema.sql"
    queries: "users.sql"
    gen:
      go:
        package: "db"
        out: "."
```

---

## 3️⃣ Config (`config/config.go`)

```go
package config

import "os"

func DBUrl() string {
    return os.Getenv("DATABASE_URL")
}
```

---

## 4️⃣ Logger (`internal/logger/logger.go`)

```go
package logger

import "go.uber.org/zap"

func New() *zap.Logger {
    log, _ := zap.NewProduction()
    return log
}
```

---

## 5️⃣ Model (`internal/models/user.go`)

```go
package models

import "time"

type User struct {
    ID   int64     `json:"id"`
    Name string    `json:"name" validate:"required"`
    DOB  time.Time `json:"dob" validate:"required"`
    Age  int       `json:"age,omitempty"`
}
```

---

## 6️⃣ Service (`internal/service/age.go`)

```go
package service

import "time"

func CalculateAge(dob time.Time) int {
    now := time.Now()
    age := now.Year() - dob.Year()
    if now.YearDay() < dob.YearDay() {
        age--
    }
    return age
}
```

---

## 7️⃣ User Service (`internal/service/user_service.go`)

```go
package service

import (
    "context"
    "github.com/yourusername/user-api/internal/models"
    "github.com/yourusername/user-api/internal/repository"
)

type UserService struct {
    repo *repository.UserRepository
}

func New(repo *repository.UserRepository) *UserService {
    return &UserService{repo}
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*models.User, error) {
    u, err := s.repo.Q.GetUser(ctx, id)
    if err != nil {
        return nil, err
    }
    return &models.User{ID: u.ID, Name: u.Name, DOB: u.Dob, Age: CalculateAge(u.Dob)}, nil
}
```

---

## 8️⃣ Repository (`internal/repository/user_repository.go`)

```go
package repository

import "github.com/yourusername/user-api/db/sqlc"

type UserRepository struct {
    Q *db.Queries
}

func New(q *db.Queries) *UserRepository {
    return &UserRepository{Q: q}
}
```

---

## 9️⃣ Middleware (`internal/middleware/request_logger.go`)

```go
package middleware

import (
    "time"
    "github.com/gofiber/fiber/v2"
    "go.uber.org/zap"
)

func RequestLogger(log *zap.Logger) fiber.Handler {
    return func(c *fiber.Ctx) error {
        start := time.Now()
        err := c.Next()
        log.Info("request",
            zap.String("path", c.Path()),
            zap.Duration("duration", time.Since(start)),
        )
        return err
    }
}
```

---

## 🔟 Handler, Routes, Server (`cmd/server/main.go`)

```go
package main

import (
    "context"
    "github.com/gofiber/fiber/v2"
    "github.com/jackc/pgx/v5"
    "github.com/yourusername/user-api/config"
    "github.com/yourusername/user-api/db/sqlc"
    "github.com/yourusername/user-api/internal/handler"
    "github.com/yourusername/user-api/internal/logger"
    "github.com/yourusername/user-api/internal/middleware"
    "github.com/yourusername/user-api/internal/repository"
    "github.com/yourusername/user-api/internal/routes"
    "github.com/yourusername/user-api/internal/service"
)

func main() {
    log := logger.New()
    conn, _ := pgx.Connect(context.Background(), config.DBUrl())
    q := db.New(conn)

    repo := repository.New(q)
    svc := service.New(repo)
    h := handler.New(svc)

    app := fiber.New()
    app.Use(middleware.RequestLogger(log))
    routes.Setup(app, h)

    app.Listen(":8080")
}
```

---


##  Features

- Create, read, update, delete users
- Store `dob` in database and calculate `age` dynamically
- RESTful API design
- SQLC-generated database layer (no raw SQL in handlers)
- Input validation using `go-playground/validator`
- Structured logging with Uber Zap
- Clean folder structure and separation of concerns

---

## 🧱 Tech Stack

- **Language:** Go (1.22+)
- **Framework:** GoFiber
- **Database:** PostgreSQL
- **ORM / DB Layer:** SQLC
- **Validation:** go-playground/validator
- **Logging:** Uber Zap

---

## 📁 Project Structure

```


````

---

## ⚙️ Setup Instructions

### 1️⃣ Prerequisites

- Go 1.22+
- PostgreSQL
- SQLC (`go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest`)

---

### 2️⃣ Database Setup

Create a PostgreSQL database and export the connection URL:

```bash
export DATABASE_URL=postgres://user:password@localhost:5432/userdb
````

Run the migration:

```sql
-- db/migrations/001_create_users.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    dob DATE NOT NULL
);
```

---

### 3️⃣ Generate SQLC Code

```bash
sqlc generate
```

---

### 4️⃣ Run the Application

```bash
go run cmd/server/main.go
```

Server will start on:

```
http://localhost:8080
```

---

## 🔌 API Endpoints

### ➕ Create User

**POST** `/users`

```json
{
  "name": "Alice",
  "dob": "1990-05-10"
}
```

---

### 📄 Get User by ID

**GET** `/users/:id`

```json
{
  "id": 1,
  "name": "Alice",
  "dob": "1990-05-10",
  "age": 35
}
```

---

### ✏️ Update User

**PUT** `/users/:id`

```json
{
  "name": "Alice Updated",
  "dob": "1991-03-15"
}
```

---

### ❌ Delete User

**DELETE** `/users/:id`

Returns `204 No Content`

---

### 📃 List All Users

**GET** `/users`

```json
[
  {
    "id": 1,
    "name": "Alice",
    "dob": "1990-05-10",
    "age": 34
  }
]
```

---

## 🧠 Age Calculation Logic

Age is calculated dynamically using Go’s `time` package:

* Current year – birth year
* Adjusted if birthday hasn’t occurred yet this year

This ensures the age is always accurate without storing it in the database.

---

## 🧪 Validation & Error Handling

* Request payloads are validated using `go-playground/validator`
* Proper HTTP status codes are returned (`400`, `404`, `500`, etc.)
* Centralized logging for easier debugging

---

