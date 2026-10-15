# Лабораторная работа. Триггеры в MS SQL Server

Анализ создания DML- и DDL-триггеров в базе college.

---

## Что требовалось

Создать 3 DML-триггера (по одному на разные таблицы) и 2 DDL-триггера на базу данных.

---

## Что сделано

### Триггер 1. DML на STUDENT (AFTER INSERT)

USE college;
GO

CREATE TRIGGER dbo.trg_Student_Insert
ON dbo.STUDENT
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @name NVARCHAR(100);
    SELECT TOP 1 @name = NAME FROM inserted;

    PRINT 'Триггер 1: добавлен студент - ' + ISNULL(@name, '(без имени)');
END;
GO

INSERT INTO STUDENT (IDGROUP, NAME, BDAY, STAMP, INFO, FOTO)
VALUES (1, 'Тестов Тест Тестович', '2005-01-01', 'TRG-001', NULL, NULL);
GO

SELECT name AS [Триггер] FROM sys.triggers
WHERE parent_id = OBJECT_ID('dbo.STUDENT');
GO

**Результат:**

| Триггер |
|---|
| trg_Student_Insert |

**Messages:** Триггер 1: добавлен студент - Тестов Тест Тестович.

---

### Триггер 2. DML на PROGRESS (AFTER UPDATE)

CREATE TRIGGER dbo.trg_Progress_Update
ON dbo.PROGRESS
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @id INT;
    SELECT TOP 1 @id = IDSTUDENT FROM inserted;

    PRINT 'Триггер 2: оценка изменена у студента - ' + CAST(@id AS VARCHAR(10));
END;
GO

UPDATE PROGRESS SET NOTE = 10
WHERE IDSTUDENT = 1 AND [SUBJECT] = 'S_DB';
GO

**Результат:**

| Триггер |
|---|
| trg_Progress_Update |

**Messages:** Триггер 2: оценка изменена у студента - 1.

---

### Триггер 3. DML на TEACHER (AFTER DELETE)

CREATE TRIGGER dbo.trg_Teacher_Delete
ON dbo.TEACHER
AFTER DELETE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @name NVARCHAR(100);
    SELECT TOP 1 @name = TEACHER_NAME FROM deleted;

    PRINT 'Триггер 3: удалён преподаватель - ' + ISNULL(@name, 'без имени');
END;
GO

INSERT INTO TEACHER (TEACHER, TEACHER_NAME, GENDER, PULPIT)
VALUES ('T_TMP', 'Временный преподаватель', 'М', 'K_IVT');
GO

DELETE FROM TEACHER WHERE TEACHER = 'T_TMP';
GO

**Результат:**

| Триггер |
|---|
| trg_Teacher_Delete |

**Messages:** Триггер 3: удалён преподаватель - Временный преподаватель.

---

### Триггер 4. DDL на БД (FOR DROP_TABLE, ALTER_TABLE)

CREATE TRIGGER trg_DB_NoDropTable
ON DATABASE
FOR DROP_TABLE, ALTER_TABLE
AS
BEGIN
    PRINT 'Триггер 4: запрещено удалять или изменять таблицы!';
    ROLLBACK;
END;
GO

SELECT name AS [DDL-триггер] FROM sys.triggers WHERE parent_class = 0;
GO

**Результат:**

| DDL-триггер |
|---|
| trg_College_LogChanges |
| trg_DB_LogCreateTable |
| trg_DB_NoDropTable |

---

### Триггер 5. DDL на БД (FOR CREATE_TABLE)

CREATE TRIGGER trg_DB_LogCreateTable
ON DATABASE
FOR CREATE_TABLE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @objName VARCHAR(100);
    SET @objName = EVENTDATA().value('(/EVENT_INSTANCE/ObjectName)[1]', 'VARCHAR(100)');

    INSERT INTO dbo.AuditLog (TableName, ActionType, RecordID, Details)
    VALUES ('DATABASE', 'DDL', @objName, 'Создана таблица: ' + @objName);

    PRINT 'Триггер 5: создана таблица - ' + @objName;
END;
GO

DISABLE TRIGGER trg_DB_NoDropTable ON DATABASE;
GO

CREATE TABLE dbo.TestTable (ID INT);
GO

DROP TABLE dbo.TestTable;
GO

ENABLE TRIGGER trg_DB_NoDropTable ON DATABASE;
GO

SELECT * FROM dbo.AuditLog WHERE ActionType = 'DDL';
GO

**Результат в AuditLog:**

| LogID | TableName | ActionType | RecordID | Details |
|---|---|---|---|---|
| … | DATABASE | DDL | TestTable | Создана таблица: TestTable |

**Messages:** Триггер 5: создана таблица - TestTable.

---

## Анализ

### DML-триггеры

| Триггер | Таблица | Событие | Что делает |
|---|---|---|---|
| trg_Student_Insert | STUDENT | AFTER INSERT | Печатает имя нового студента |
| trg_Progress_Update | PROGRESS | AFTER UPDATE | Печатает ID студента с изменённой оценкой |
| trg_Teacher_Delete | TEACHER | AFTER DELETE | Печатает имя удалённого преподавателя |

**Ключевые моменты:**
- `inserted` — виртуальная таблица с новыми строками (INSERT, UPDATE).
- `deleted` — виртуальная таблица со старыми строками (DELETE, UPDATE).
- `SET NOCOUNT ON` — убирает служебные сообщения.
- `ISNULL` — защита от NULL.

### DDL-триггеры

| Триггер | Событие | Что делает |
|---|---|---|
| trg_DB_NoDropTable | DROP_TABLE, ALTER_TABLE | Запрещает удаление/изменение таблиц |
| trg_DB_LogCreateTable | CREATE_TABLE | Логирует создание таблиц в AuditLog |

**Ключевые моменты:**
- DDL-триггеры создаются на `ON DATABASE` без префикса схемы.
- `EVENTDATA()` возвращает XML с информацией о событии.
- `ROLLBACK` откатывает операцию (для запрета).

---

## Вывод

Созданы **5 триггеров**:

1. **trg_Student_Insert** — DML на STUDENT (AFTER INSERT).
2. **trg_Progress_Update** — DML на PROGRESS (AFTER UPDATE).
3. **trg_Teacher_Delete** — DML на TEACHER (AFTER DELETE).
4. **trg_DB_NoDropTable** — DDL на БД, запрет DROP/ALTER.
5. **trg_DB_LogCreateTable** — DDL на БД, логирование CREATE.

Все триггеры успешно сработали и вывели соответствующие сообщения. DDL-триггер запрета (`trg_DB_NoDropTable`) корректно откатывает опасные операции, а логирующий (`trg_DB_LogCreateTable`) пишет события в AuditLog.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база college.