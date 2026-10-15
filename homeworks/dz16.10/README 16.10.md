# Лабораторная работа. Логины, пользователи, роли и разрешения

Анализ задач 1–13 по работе с логинами, пользователями, ролями и разрешениями
в базе данных college.

---

## Задание 1. Создание SQL-логина с политикой паролей

### Что требовалось
Создать SQL-логин TestUser01 с паролем P@ssw0rd123!, CHECK_POLICY = ON, CHECK_EXPIRATION = OFF. Проверить через sys.sql_logins.

### Что сделано

CREATE LOGIN TestUser01
WITH PASSWORD = 'P@ssw0rd123!',
     CHECK_POLICY = ON,
     CHECK_EXPIRATION = OFF,
     DEFAULT_DATABASE = master;

### Результат

Логин | Тип | Проверка политики | Проверка срока | База
TestUser01 | SQL_LOGIN | 1 | 0 | master

### Анализ
is_policy_checked = 1 — политика включена. Пароль соответствует: длина 12, есть заглавные, строчные, цифры, спецсимволы.

### Вывод
Логин создан с соблюдением политики сложности паролей. CHECK_POLICY = ON — обязательное требование безопасности.

---

## Задание 2. Создание логина Windows и сопоставление с пользователем БД

### Что требовалось
Создать логин для Windows-учётки DESKTOP-7JGCBJA\Edgar. Создать пользователя Edgar в базе college, привязанного к логину. Добавить в db_datareader.

### Что сделано

CREATE LOGIN [DESKTOP-7JGCBJA\Edgar]
FROM WINDOWS WITH DEFAULT_DATABASE = college;

USE college;
CREATE USER Edgar FOR LOGIN [DESKTOP-7JGCBJA\Edgar];
ALTER ROLE db_datareader ADD MEMBER Edgar;

### Результат

Логин | Тип | База
DESKTOP-7JGCBJA\Edgar | WINDOWS_LOGIN | college

Пользователь | Тип | Схема
Edgar | SQL_USER | dbo

Роль | Пользователь
db_datareader | Edgar

### Анализ
Логин — на уровне сервера, пользователь — на уровне базы. Один логин может иметь разных пользователей в разных базах.

### Вывод
Windows-логин успешно сопоставлен с пользователем БД. Разделение логин/пользователь позволяет управлять правами в разных базах.

---

## Задание 3. Создание пользователя без логина

### Что требовалось
Создать пользователя AppServiceUser WITHOUT LOGIN в базе college. Проверить, что он не может подключиться. Продемонстрировать EXECUTE AS USER.

### Что сделано

CREATE USER AppServiceUser WITHOUT LOGIN
WITH DEFAULT_SCHEMA = dbo;

ALTER ROLE db_datareader ADD MEMBER AppServiceUser;

EXECUTE AS USER = 'AppServiceUser';
SELECT USER_NAME();
REVERT;

### Результат

Пользователь | Тип | Схема
AppServiceUser | SQL_USER | dbo

Логин на сервере — пусто.

EXECUTE AS:
До — dbo
После EXECUTE AS — AppServiceUser
После REVERT — dbo

### Анализ
WITHOUT LOGIN создаёт пользователя только в базе. Он не может подключаться к серверу. Используется для EXECUTE AS, подписи модулей, владения схемами.

### Вывод
Такой пользователь — удобный контейнер прав. Ограничение — невозможность прямого подключения.

---

## Задание 4. Создание логина с истёкшим сроком действия пароля

### Что требовалось
Создать логин TempUser с CHECK_EXPIRATION = ON, CHECK_POLICY = ON. Пароль — MUST_CHANGE. Проверить через LOGINPROPERTY.

### Что сделано

CREATE LOGIN TempUser
WITH PASSWORD = 'P@ssw0rd123!',
     CHECK_POLICY = ON,
     CHECK_EXPIRATION = ON;

ALTER LOGIN TempUser
WITH PASSWORD = 'P@ssw0rd123!' MUST_CHANGE;

SELECT
    name, is_policy_checked, is_expiration_checked,
    LOGINPROPERTY(name, 'IsMustChange') AS [Требует смены],
    LOGINPROPERTY(name, 'IsExpired') AS [Истёк],
    LOGINPROPERTY(name, 'IsLocked') AS [Заблокирован],
    LOGINPROPERTY(name, 'PasswordLastSetTime') AS [Пароль установлен]
FROM sys.sql_logins WHERE name = 'TempUser';

### Результат

Логин | Политика | Срок | Требует смены | Истёк | Заблокирован
TempUser | 1 | 1 | 1 | 0 | 0

### Анализ
IsMustChange = 1 — имитация истёкшего пароля. При входе SQL Server вернёт Msg 18488: The password must be changed before logging in the first time.

### Вывод
MUST_CHANGE даёт тот же эффект, что истёкший пароль — вход невозможен без смены. Реальное истечение (42 дня) воспроизвести сложно.

---

## Задание 5. Просмотр списка всех логинов и пользователей

### Что требовалось
Вывести все логины сервера и всех пользователей базы college.

### Что сделано

USE master;
SELECT name, type_desc, create_date, modify_date, is_disabled, default_database_name
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY name;

USE college;
SELECT name, type_desc, create_date, modify_date, default_schema_name
FROM sys.database_principals
WHERE type IN ('S', 'U', 'G')
  AND name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
ORDER BY name;

### Результат
Логины: sa, dev1, Edgar7777\Babay, NT AUTHORITY\SYSTEM, spfuser, TempUser, TestUser01 и др.
Пользователи college: AppServiceUser.

### Анализ
type IN ('S','U','G') отсекает системные роли. is_disabled показывает отключённые логины.

### Вывод
Запросы дают полный обзор учётных записей сервера и базы.

---

## Задание 6. Добавление логина в фиксированную серверную роль

### Что требовалось
Добавить TestUser01 в securityadmin. Проверить членство.

### Что сделано

ALTER SERVER ROLE securityadmin ADD MEMBER TestUser01;

SELECT r.name, m.name
FROM sys.server_role_members rm
JOIN sys.server_principals r ON r.principal_id = rm.role_principal_id
JOIN sys.server_principals m ON m.principal_id = rm.member_principal_id
WHERE m.name = 'TestUser01';

### Результат

Роль | Логин
securityadmin | TestUser01

### Анализ
securityadmin даёт: CREATE/ALTER/DROP LOGIN, GRANT/DENY/REVOKE на уровне сервера, управление разрешениями БД, сброс паролей.

### Вывод
securityadmin — мощная роль, фактически эквивалент sysadmin по безопасности. Использовать с осторожностью.

---

## Задание 7. Создание пользовательской серверной роли

### Что требовалось
Создать CustomServerRole, дать ALTER ANY LOGIN, добавить TestUser01. Проверить через fn_my_permissions.

### Что сделано

CREATE SERVER ROLE CustomServerRole;
GRANT ALTER ANY LOGIN TO CustomServerRole;
ALTER SERVER ROLE CustomServerRole ADD MEMBER TestUser01;

EXECUTE AS LOGIN = 'TestUser01';
SELECT permission_name FROM fn_my_permissions(NULL, 'SERVER')
WHERE permission_name = 'ALTER ANY LOGIN';
REVERT;

### Результат

Роль | Член роли
CustomServerRole | TestUser01

Роль | Состояние | Право
CustomServerRole | GRANT | ALTER ANY LOGIN

Доступное право
ALTER ANY LOGIN

### Анализ
Пользовательская роль с точечным правом. TestUser01 может создавать логины.

### Вывод
Пользовательские роли позволяют выдавать конкретные права без избыточных полномочий фиксированных ролей.

---

## Задание 8. Проверка членства в серверных ролях

### Что требовалось
Для указанного логина вывести все серверные роли.

### Что сделано

SELECT sp.name, r.name, r.type_desc, r.is_fixed_role
FROM sys.server_role_members rm
JOIN sys.server_principals r ON r.principal_id = rm.role_principal_id
JOIN sys.server_principals sp ON sp.principal_id = rm.member_principal_id
WHERE sp.name = 'TestUser01'
ORDER BY r.is_fixed_role DESC, r.name;

### Результат

Логин | Роль | Тип роли | Фиксированная
TestUser01 | CustomServerRole | SERVER_ROLE | 0
TestUser01 | public | SERVER_ROLE | 1

### Анализ
is_fixed_role = 1 — фиксированная роль, 0 — пользовательская.

### Вывод
Запрос даёт полный список ролей логина, включая неявный public.

---

## Задание 9. Удаление логина из роли и удаление пользовательской роли

### Что требовалось
Удалить TestUser01 из securityadmin и CustomServerRole. Удалить CustomServerRole.

### Что сделано

ALTER SERVER ROLE securityadmin DROP MEMBER TestUser01;
ALTER SERVER ROLE CustomServerRole DROP MEMBER TestUser01;
DROP SERVER ROLE CustomServerRole;

### Результат
TestUser01 больше не в securityadmin и CustomServerRole. Роль CustomServerRole удалена.

### Условия для удаления роли
1. Роль пуста (нет членов).
2. Роль не владеет объектами.
3. Роль не фиксированная.
4. Есть право ALTER ANY SERVER ROLE.

### Вывод
Удаление роли — двухшаговая операция: очистить от членов → DROP SERVER ROLE.

---

## Задание 10. Добавление пользователя в фиксированную роль БД

### Что требовалось
Добавить JohnDoe в db_datareader в базе college. Проверить, что он может SELECT, но не INSERT/UPDATE.

### Что сделано

CREATE LOGIN JohnDoe WITH PASSWORD = 'P@ssw0rd123!', CHECK_POLICY = ON, DEFAULT_DATABASE = college;
USE college;
CREATE USER JohnDoe FOR LOGIN JohnDoe;
ALTER ROLE db_datareader ADD MEMBER JohnDoe;

EXECUTE AS USER = 'JohnDoe';
SELECT TOP 1 * FROM dbo.STUDENT;  -- работает
INSERT INTO dbo.STUDENT ...;       -- ошибка Msg 229
UPDATE dbo.STUDENT ...;            -- ошибка Msg 229
REVERT;

### Результат

Роль | Пользователь | Тип роли | Фиксированная
db_datareader | JohnDoe | DATABASE_ROLE | 1

- SELECT — успех
- INSERT — отклонён (Msg 229)
- UPDATE — отклонён (Msg 229)

### Анализ
db_datareader даёт только чтение. INSERT/UPDATE отклонены.

### Вывод
Фиксированная роль — быстрый способ дать типовые права. Принцип минимальных привилегий.

---

## Задание 11. Создание пользовательской роли с точечными правами

### Что требовалось
Создать SalesRole с правами SELECT, INSERT, UPDATE на dbo.STUDENT. Добавить JohnDoe.

### Что сделано

CREATE ROLE SalesRole;
GRANT SELECT ON dbo.STUDENT TO SalesRole;
GRANT INSERT ON dbo.STUDENT TO SalesRole;
GRANT UPDATE ON dbo.STUDENT TO SalesRole;
ALTER ROLE SalesRole ADD MEMBER JohnDoe;

EXECUTE AS USER = 'JohnDoe';
SELECT ...;  -- OK
INSERT ...;  -- OK
UPDATE ...;  -- OK
DELETE ...;  -- Msg 229
REVERT;

### Результат

Роль | Право | Состояние | Объект
SalesRole | INSERT | GRANT | STUDENT
SalesRole | SELECT | GRANT | STUDENT
SalesRole | UPDATE | GRANT | STUDENT

- SELECT — успех
- INSERT — успех
- UPDATE — успех
- DELETE — отклонён (Msg 229)

### Анализ
Точечные права дают ровно то, что нужно. DELETE не выдан — удаление невозможно.

### Вывод
Принцип минимальных привилегий. Пользовательская роль — гибкий инструмент для точной настройки прав.

---

## Задание 12. Запрет (DENY) на уровне объекта

### Что требовалось
Создать RestrictedUser. GRANT SELECT ON SCHEMA::Sales, DENY SELECT ON Sales.CreditCard. Проверить, что DENY работает.

### Что сделано

CREATE USER RestrictedUser FOR LOGIN RestrictedUser;

GRANT SELECT ON SCHEMA::Sales TO RestrictedUser;
DENY SELECT ON Sales.CreditCard TO RestrictedUser;

EXECUTE AS USER = 'RestrictedUser';
SELECT * FROM Sales.Orders;      -- OK (право через схему)
SELECT * FROM Sales.CreditCard;  -- Msg 229 (DENY)
REVERT;

### Результат

Пользователь | Право | Состояние | Уровень | Объект
RestrictedUser | SELECT | DENY | OBJECT_OR_COLUMN | CreditCard
RestrictedUser | CONNECT | GRANT | DATABASE | NULL
RestrictedUser | SELECT | GRANT | SCHEMA | Sales

- SELECT из Sales.Orders — успех
- SELECT из Sales.CreditCard — отклонён (Msg 229)

### Анализ
DENY приоритетнее GRANT. Даже при праве на всю схему явный DENY на таблицу перекрывает его.

### Вывод
Точечный DENY позволяет запретить доступ к чувствительным данным без изменения общих прав.

---

## Задание 13. Права на выполнение хранимых процедур

### Что требовалось
Дать AppServiceUser EXECUTE на uspGetEmployeeManagers. Проверить, что другие процедуры недоступны.

### Что сделано

CREATE USER AppServiceUser WITHOUT LOGIN;

CREATE PROCEDURE dbo.uspGetEmployeeManagers AS ...
CREATE PROCEDURE dbo.uspGetAllStudents AS ...

GRANT EXECUTE ON dbo.uspGetEmployeeManagers TO AppServiceUser;

EXECUTE AS USER = 'AppServiceUser';
EXEC dbo.uspGetEmployeeManagers;  -- OK
EXEC dbo.uspGetAllStudents;        -- Msg 229
REVERT;

### Результат

Пользователь | Право | Состояние | Объект
AppServiceUser | CONNECT | GRANT | NULL
AppServiceUser | EXECUTE | GRANT | uspGetEmployeeManagers

Сообщение: uspGetEmployeeManagers: только эта процедура разрешена

- EXEC uspGetEmployeeManagers — успех
- EXEC uspGetAllStudents — отклонён (Msg 229)

### Анализ
Право EXECUTE выдаётся точечно. AppServiceUser может вызывать только разрешённый код.

### Вывод
Точечные права на процедуры — стандартный приём для сервисных учёток. Пользователь не имеет доступа к таблицам напрямую, но может выполнять разрешённые операции через процедуру.

---

## Общий вывод

| Задание | Что показало |
|---|---|
| 1 | SQL-логин с CHECK_POLICY = ON |
| 2 | Windows-логин + пользователь БД |
| 3 | Пользователь без логина (WITHOUT LOGIN) |
| 4 | Логин с MUST_CHANGE (истёкший пароль) |
| 5 | Просмотр логинов и пользователей |
| 6 | securityadmin — мощная фиксированная роль |
| 7 | Пользовательская роль с ALTER ANY LOGIN |
| 8 | Проверка членства в ролях |
| 9 | Удаление из ролей и удаление роли |
| 10 | db_datareader — только чтение |
| 11 | Точечные права через пользовательскую роль |
| 12 | DENY приоритетнее GRANT |
| 13 | EXECUTE на конкретную процедуру |

**Ключевые принципы:**
1. Логин — сервер, пользователь — база.
2. CHECK_POLICY = ON — обязательное требование безопасности.
3. Фиксированные роли дают типовые права.
4. Пользовательские роли — точная настройка.
5. DENY > GRANT.
6. Минимальные привилегии — только то, что нужно.
7. WITHOUT LOGIN — для EXECUTE AS, подписи, схем.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база college.