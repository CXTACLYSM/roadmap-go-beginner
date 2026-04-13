# Модуль 3. database/sql из Go — условие

## Задание

Написать **минимальную Go-программу**, которая подключается к PostgreSQL, проверяет соединение через `Ping` и выполняет два простых запроса. Это «hello world» для работы с БД из Go.

## Что должна делать программа

```bash
$ DATABASE_URL="postgres://postgres:secret@localhost:5432/postgres?sslmode=disable" go run .
2026/04/13 12:00:00 Подключение к PostgreSQL установлено
2026/04/13 12:00:00 SELECT 1 вернул: 1
2026/04/13 12:00:00 Версия PostgreSQL: PostgreSQL 16.1 (Debian 16.1-1.pgdg120+1) on x86_64-pc-linux-gnu
```

При неправильном DSN или недоступной БД — программа должна **упасть на старте** с понятной ошибкой (через `log.Fatal`).

## Требования

1. Создан подпроект (например, в папке `practice-db/` внутри `005-todo-postgres/`) с собственным `go.mod`
2. Подключён драйвер: `go get github.com/jackc/pgx/v5/stdlib`
3. Импортирован драйвер через `_ "github.com/jackc/pgx/v5/stdlib"`
4. **DSN читается из переменной окружения `DATABASE_URL`**, не хардкодится
5. Если переменная не задана — программа выходит через `log.Fatal` с понятным сообщением
6. Подключение через `sql.Open("pgx", dsn)`
7. Вызывается `db.Ping()` сразу после `Open`
8. При ошибке `Ping` — `log.Fatalf` с понятным сообщением
9. Выполнен запрос `SELECT 1`, результат записан в `int` через `Scan`
10. Выполнен запрос `SELECT version()`, результат записан в `string` через `Scan`
11. `defer db.Close()` присутствует
12. Используется пакет `log` (не `fmt`) для всех сообщений

## Шаги

1. Создай папку `practice-db/`, инициализируй модуль:
   ```bash
   mkdir practice-db && cd practice-db
   go mod init practice-db
   ```

2. Установи драйвер:
   ```bash
   go get github.com/jackc/pgx/v5/stdlib
   ```

3. Создай `main.go` со скелетом:
   ```go
   package main

   import (
       "database/sql"
       "log"
       "os"

       _ "github.com/jackc/pgx/v5/stdlib"
   )

   func main() {
       // твой код
   }
   ```

4. Прочитай `DATABASE_URL` через `os.Getenv`
5. Проверь, что не пустая
6. Вызови `sql.Open`, обработай ошибку
7. Добавь `defer db.Close()`
8. Вызови `db.Ping()`, обработай ошибку
9. Выполни `SELECT 1` через `QueryRow().Scan()`
10. Выполни `SELECT version()` через `QueryRow().Scan()`
11. Запусти:
    ```bash
    DATABASE_URL="postgres://postgres:secret@localhost:5432/postgres?sslmode=disable" go run .
    ```
12. Должно вывести три строки с информацией

## Эксперименты

1. **Запусти без `DATABASE_URL`** — увидишь сообщение и выход через `log.Fatal`. Это хорошее поведение: программа не «работает наполовину».

2. **Запусти с неправильным паролем:**
   ```bash
   DATABASE_URL="postgres://postgres:WRONG@localhost:5432/postgres?sslmode=disable" go run .
   ```
   Какое сообщение? Где оно появляется — в `Open` или в `Ping`? Это и есть **причина**, почему `Ping` обязателен: иначе ты бы узнал об ошибке только при первом полезном запросе.

3. **Запусти при остановленном контейнере:**
   ```bash
   docker stop todo-pg
   DATABASE_URL="postgres://postgres:secret@localhost:5432/postgres?sslmode=disable" go run .
   ```
   Какое сообщение? Это `connection refused` — БД не отвечает на порту.

4. **Запусти без `?sslmode=disable`:**
   ```bash
   DATABASE_URL="postgres://postgres:secret@localhost:5432/postgres" go run .
   ```
   Что произошло? Если контейнер настроен по умолчанию — может работать (pgx иногда сам угадывает). Если нет — получишь ошибку про SSL. Привычка `?sslmode=disable` для локальной разработки — стандарт.

5. **Запусти на несуществующую БД:**
   ```bash
   DATABASE_URL="postgres://postgres:secret@localhost:5432/nosuchdb?sslmode=disable" go run .
   ```
   Какое сообщение? `database "nosuchdb" does not exist`. БД нужно создавать заранее (`CREATE DATABASE` или через `psql`).

6. **Забудь импорт драйвера:**
   ```go
   // import _ "github.com/jackc/pgx/v5/stdlib"  // закомментируй
   ```
   Что произошло? Должно быть `sql: unknown driver "pgx" (forgotten import?)`. Это и есть смысл импорта ради побочного эффекта — без него драйвер не зарегистрирован.

7. **Добавь настройки пула:**
   ```go
   db.SetMaxOpenConns(10)
   db.SetMaxIdleConns(2)
   db.SetConnMaxLifetime(5 * time.Minute)
   ```
   (Не забудь импорт `"time"`.) Программа всё ещё работает? Эти настройки не меняют поведение при одиночных запросах, но в нагруженном сервере они критичны.

8. **Сделай несколько Ping подряд:**
   ```go
   for i := 0; i < 5; i++ {
       if err := db.Ping(); err != nil {
           log.Printf("ping %d failed: %v", i, err)
       } else {
           log.Printf("ping %d OK", i)
       }
   }
   ```
   Проверь, что все 5 успешные. Это **проверка стабильности соединения** — иногда полезно для health checks.

9. **Подключись к таблице, которая создавалась в модулях 1-2:**
   ```go
   var count int
   if err := db.QueryRow("SELECT COUNT(*) FROM tasks").Scan(&count); err != nil {
       log.Fatalf("count tasks: %v", err)
   }
   log.Printf("Задач в БД: %d", count)
   ```
   Это первый запрос «по делу» к **твоим** данным. В модуле 4 ты сделаешь полный CRUD на этом основании.

## Что показать Claude

Покажи Claude свою программу и спроси: «правильно ли я обрабатываю ошибки? Стоит ли мне добавить `context.Context` к запросам?». Ответ — **да, в продакшене**: контексты позволяют отменять запросы при таймаутах или отмене HTTP-запроса. Сейчас, для модуля 3, упрощённая версия без контекста — нормально.

В модуле 4 ты познакомишься с `db.QueryContext` и `db.QueryRowContext`. Запиши себе на будущее: **в реальных серверах всегда используют контексты**, а версии без контекста (`Query`, `QueryRow`) — только в скриптах и учебных примерах.
