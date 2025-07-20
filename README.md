# Система аутентификации с поддержкой OTP на Java

## Описание проекта

Данное приложение реализует полноценную систему регистрации и аутентификации пользователей с поддержкой ролей, генерации и проверки одноразовых паролей (OTP). Проект построен на Java и использует PostgreSQL для хранения данных.

Особенности:

- Регистрация и вход пользователей с проверкой пароля.
- Роли пользователей: **USER** и **ADMIN**, с ограничениями на количество администраторов.
- Аутентификация через токены с ограниченным временем жизни.
- Поддержка генерации OTP с гибкими настройками длины и времени жизни.
- Отправка OTP через несколько каналов: файл, email, SMS, Telegram.
- REST-подобный API с JSON-запросами и ответами.
- Легковесный HTTP-сервер из стандартного JDK.

---

## Содержание

- [Требования](#требования)
- [Установка и запуск](#установка-и-запуск)
- [Описание API](#описание-api)
- [Примеры запросов и ответов](#примеры-запросов-и-ответов)
- [Структура проекта и модули](#структура-проекта-и-модули)
- [Внешние библиотеки и интеграции](#внешние-библиотеки-и-интеграции)
- [Тестирование](#тестирование)
- [Рекомендации по безопасности](#рекомендации-по-безопасности)

---

## Требования

- **Java 11+** (JDK)
- **PostgreSQL 9.6+**
- Maven (рекомендуется для сборки, но можно использовать любой способ компиляции)
- Подключение к интернету для загрузки зависимостей (при использовании Maven)

---

## Установка и запуск

### 1. Настройка базы данных PostgreSQL

Выполните следующие команды в вашей базе:

CREATE TABLE users (
id SERIAL PRIMARY KEY,
login VARCHAR(255) UNIQUE NOT NULL,
password_hash VARCHAR(255) NOT NULL,
role VARCHAR(50) NOT NULL
);

CREATE TABLE otp_codes (
id SERIAL PRIMARY KEY,
user_id INT REFERENCES users(id),
operation_id VARCHAR(255),
code VARCHAR(255),
status VARCHAR(50),
created_at TIMESTAMP
);

CREATE TABLE otp_config (
id INT PRIMARY KEY DEFAULT 1,
code_length INT,
ttl_seconds INT
);

INSERT INTO otp_config (id, code_length, ttl_seconds) VALUES (1, 6, 60);

> *Измените параметры подключения к базе в `Database.java` при необходимости.*

### 2. Сборка и запуск

Соберите проект (если используется Maven):

mvn clean package


Запустите:

java -jar target/yourapp.jar

---

## Описание API

| Метод | URL                 | Описание                                 | Требования                                 | Роль  |
|-------|---------------------|------------------------------------------|-------------------------------------------|-------|
| POST  | `/register`         | Регистрация пользователя                  | JSON с `login`, `password`, `role`        | -     |
| POST  | `/login`            | Логин, получение токена                   | JSON с `login`, `password`                 | -     |
| GET   | `/admin/config`     | Получение конфигурации OTP                | Заголовок `Authorization` с токеном       | ADMIN |
| POST  | `/admin/config`     | Обновление конфигурации OTP               | JSON с `code_length`, `ttl_seconds`       | ADMIN |
| GET   | `/admin/users`      | Получить список пользователей (кроме админов) | Заголовок `Authorization`                | ADMIN |
| POST  | `/admin/delete`     | Удаление пользователя                     | JSON с `user_id`                           | ADMIN |
| POST  | `/user/gen-otp`     | Генерация OTP                            | JSON с `operation_id`, `channel`, и параметры канала | USER  |
| POST  | `/user/validate-otp`| Проверка OTP                            | JSON с `operation_id`, `otp_code`         | USER  |

---

## Примеры запросов и ответов

### Регистрация пользователя

curl -X POST http://localhost:8082/register
-H "Content-Type: application/json"
-d '{"login":"user1","password":"pass1234","role":"USER"}'

**Ответ:**
{"result":"ok"}
### Логин

curl -X POST http://localhost:8082/login
-H "Content-Type: application/json"
-d '{"login":"user1","password":"pass1234"}'


**Ответ:**

{"token":"123e4567-e89b-12d3-a456-426614174000"}


### Генерация OTP (файл)

curl -X POST http://localhost:8082/user/gen-otp
-H "Content-Type: application/json"
-H "Authorization: 123e4567-e89b-12d3-a456-426614174000"
-d '{"operation_id":"payment"}'


**Ответ:**

{"result":"ok"}


### Проверка OTP

curl -X POST http://localhost:8082/user/validate-otp
-H "Content-Type: application/json"
-H "Authorization: 123e4567-e89b-12d3-a456-426614174000"
-d '{"operation_id":"payment","otp_code":"123456"}'


**Ответ:**

{"valid":true}


---

## Структура проекта и описание модулей

### Database

- Управление SQL-запросами и подключением к PostgreSQL
- CRUD операции пользователя и OTP
- Конфигурация OTP

### AuthService

- Бизнес-логика авторизации и токенов
- Генерация и проверка OTP
- Проверка ролей и прав доступа

### ApiHandlers

- Обработчики HTTP-запросов
- Парсинг JSON-запросов и формирование ответов
- Вызов логики из AuthService и Database

### NotificationService

- Отправка OTP через выбранные каналы (файлы, email, SMS, Telegram)
- Поддержка добавления новых каналов

---

## Внешние библиотеки и установка

- **PostgreSQL JDBC драйвер**  
  [https://jdbc.postgresql.org/download.html](https://jdbc.postgresql.org/download.html)  
  (или через Maven с координатами ниже)

- **Maven зависимости (пример):**

<dependency> <groupId>org.postgresql</groupId> <artifactId>postgresql</artifactId> <version>42.6.0</version> </dependency> <dependency> <groupId>org.slf4j</groupId> <artifactId>slf4j-api</artifactId> <version>2.0.7</version> </dependency> <dependency> <groupId>ch.qos.logback</groupId> <artifactId>logback-classic</artifactId> <version>1.4.6</version> </dependency> ```
Для отправки email, SMS и Telegram настраивайте соответствующие API и библиотеки.

### Настройка каналов доставки OTP
###Telegram Bot
Для отправки OTP через Telegram необходимо зарегистрировать бота и получить токен:

В Telegram найдите и запустите бота BotFather.

Создайте нового бота командой /newbot.

Следуйте инструкциям и получите уникальный токен доступа к боту.

Этот токен вставьте в конфигурацию вашего приложения (например, в NotificationService).

Обратите внимание: чтобы бот мог отправлять сообщения пользователям, они должны начать диалог с этим ботом или быть в группе с ботом.


### Настройка SMTP через Яндекс.Почту
Для отправки email с OTP через SMTP Яндекса:

Войдите в вашу Яндекс.Почту.

Перейдите в настройки почтового ящика → Настройки для программы почты (IMAP/SMTP).

Включите доступ к почте через SMTP.

Создайте пароль для приложения для SMTP.

В конфигурации вашего приложения укажите следующие настройки SMTP:

SMTP-сервер: smtp.yandex.ru
Порт: 465 (SSL) или 587 (TLS)
Логин: ваш_логин_яндекса@ya.ru
Пароль: пароль_для_приложения

### Тестирование API
Используйте Postman или curl для отправки HTTP-запросов.

Важно: Для всех защищённых эндпоинтов необходимо передавать заголовок:


Authorization: <токен_пользователя>
Токен выдаётся при успешном входе в систему (/login).

При возникновении ошибок проверяйте логи сервера — они содержат подробную информацию о проблемах и помогут в их устранении.
