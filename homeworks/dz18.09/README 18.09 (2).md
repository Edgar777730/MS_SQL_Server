# Лабораторная работа. Функция (встроенные функции) и процедура (разница поступивших/выпустившихся)

Анализ задачи: создание функции с примерами встроенных функций SQL Server
и процедуры, считающей разницу между выпустившимися и поступившими студентами.

---

## Что требовалось

Создать:
1. **Функцию**, которая показывает пример всех встроенных функций SQL Server.
2. **Процедуру**, которая считает разницу между выпускающимися и поступившими студентами.

В дз отправить скрин с кодом и результатом через репозиторий.

---

## Что сделано

### 1. Функция fn_BuiltInFunctionsDemo

Создана скалярная функция, которая принимает строку и число, а возвращает строку с примерами встроенных функций SQL Server.

CREATE OR ALTER FUNCTION dbo.fn_BuiltInFunctionsDemo (@text VARCHAR(100), @num DECIMAL(10,2))
RETURNS VARCHAR(500)
AS
BEGIN
    DECLARE @result VARCHAR(500);

    SET @result =
        'LEN(text)          = ' + CAST(LEN(@text) AS VARCHAR(10))      + '; ' +
        'UPPER(text)        = ' + UPPER(@text)                         + '; ' +
        'LOWER(text)        = ' + LOWER(@text)                         + '; ' +
        'REVERSE(text)      = ' + REVERSE(@text)                       + '; ' +
        'LEFT(text, 5)      = ' + LEFT(@text, 5)                       + '; ' +
        'RIGHT(text, 5)     = ' + RIGHT(@text, 5)                      + '; ' +
        'SUBSTRING(text,2,3)= ' + SUBSTRING(@text, 2, 3)               + '; ' +
        'ROUND(num, 1)      = ' + CAST(ROUND(@num, 1) AS VARCHAR(20))  + '; ' +
        'ABS(num)           = ' + CAST(ABS(@num) AS VARCHAR(20))       + '; ' +
        'CEILING(num)       = ' + CAST(CEILING(@num) AS VARCHAR(20))   + '; ' +
        'FLOOR(num)         = ' + CAST(FLOOR(@num) AS VARCHAR(20))     + '; ' +
        'SQRT(ABS(num))     = ' + CAST(SQRT(ABS(@num)) AS VARCHAR(20)) + '; ' +
        'GETDATE()          = ' + CONVERT(VARCHAR(30), GETDATE(), 120) + '; ' +
        'YEAR(GETDATE())    = ' + CAST(YEAR(GETDATE()) AS VARCHAR(10)) + '; ' +
        'MONTH(GETDATE())   = ' + CAST(MONTH(GETDATE()) AS VARCHAR(10))+ '; ' +
        'DAY(GETDATE())     = ' + CAST(DAY(GETDATE()) AS VARCHAR(10));

    RETURN @result;
END;
GO

SELECT dbo.fn_BuiltInFunctionsDemo('SQL Server', -16.75) AS [Демонстрация встроенных функций];
GO

### 2. Процедура usp_StudentsInOutDiff

Создана процедура, которая для каждого года поступления считает:
- сколько студентов поступило;
- сколько выпустилось (поступивших 4 года назад);
- разницу.

CREATE OR ALTER PROCEDURE dbo.usp_StudentsInOutDiff
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        G.YEAR_FIRST         AS [Год поступления],
        COUNT(S.IDSTUDENT)   AS [Поступило],
        ISNULL((
            SELECT COUNT(*)
            FROM STUDENT S2
            JOIN GROUPS G2 ON G2.IDGROUP = S2.IDGROUP
            WHERE G2.YEAR_FIRST = G.YEAR_FIRST - 4
        ), 0)                AS [Выпустилось],
        COUNT(S.IDSTUDENT) - ISNULL((
            SELECT COUNT(*)
            FROM STUDENT S2
            JOIN GROUPS G2 ON G2.IDGROUP = S2.IDGROUP
            WHERE G2.YEAR_FIRST = G.YEAR_FIRST - 4
        ), 0)                AS [Разница]
    FROM GROUPS G
    LEFT JOIN STUDENT S ON S.IDGROUP = G.IDGROUP
    GROUP BY G.YEAR_FIRST
    ORDER BY G.YEAR_FIRST;
END;
GO

EXEC dbo.usp_StudentsInOutDiff;
GO

---

## Результат

### Функция fn_BuiltInFunctionsDemo

| Демонстрация встроенных функций |
|---|
| LEN(text) = 10; UPPER(text) = SQL SERVER; LOWER(text) = sql server; REVERSE(text) = revreS LQS; LEFT(text, 5) = SQL S; RIGHT(text, 5) = erver; SUBSTRING(text,2,3)= QL ; ROUND(num, 1) = -16.80; ABS(num) = 16.75; CEILING(num) = -16; FLOOR(num) = -17; SQRT(ABS(num)) = 4.09267; GETDATE() = 2026-09-18 …; YEAR = 2026; MONTH = 9; DAY = 18 |

### Процедура usp_StudentsInOutDiff

| Год поступления | Поступило | Выпустилось | Разница |
|---|---|---|---|
| 2021 | 1 | 0 | 1 |
| 2022 | 3 | 0 | 3 |
| 2023 | 5 | 0 | 5 |

---

## Анализ

### Функция

- **Скалярная функция** — возвращает одно значение (`VARCHAR(500)`).
- **CONCAT через +** — собирает строку со значениями встроенных функций.
- **CAST** — приводит числа к строке для конкатенации.
- **Демонстрирует 16 встроенных функций**: строковые (`LEN`, `UPPER`, `LOWER`, `REVERSE`, `LEFT`, `RIGHT`, `SUBSTRING`), числовые (`ROUND`, `ABS`, `CEILING`, `FLOOR`, `SQRT`), даты (`GETDATE`, `YEAR`, `MONTH`, `DAY`).

### Процедура

- **Set-based подход** — без курсора, всё через `GROUP BY`.
- **Коррелированный подзапрос** — для каждого года поступления считает выпустившихся (поступивших 4 года назад).
- **ISNULL** — если выпустившихся нет, возвращает 0.
- **LEFT JOIN** — сохраняет группы без студентов.
- **Разница** = Поступило − Выпустилось.

### Почему такой подход

- Курсор был бы медленнее.
- Один `SELECT` с `GROUP BY` — эффективнее.
- Условие `G2.YEAR_FIRST = G.YEAR_FIRST - 4` — модель 4-летнего бакалавриата.

---

## Вывод

**Функция** демонстрирует работу 16 встроенных функций SQL Server — строковых, числовых, дат. Возвращает одно строковое значение с результатами.

**Процедура** считает статистику по годам: сколько поступило, сколько выпустилось, разница. Использует set-based подход (без курсора), коррелированный подзапрос и LEFT JOIN. Выводит таблицу с тремя колонками.

**Оба объекта** созданы через `CREATE OR ALTER` — пересоздаются без ошибок при повторном запуске.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база college.