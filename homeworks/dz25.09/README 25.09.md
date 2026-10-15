# Лабораторная работа. Представления (VIEW) в MS SQL Server

Анализ создания трёх представлений в базе college:
одно модифицируемое, два аналитических.

---

## Что требовалось

Реализовать 3 представления по своей БД. Одно из них должно быть модифицируемым.

---

## Что сделано

### 1. Модифицируемое представление View_StudentsSimple (одна таблица)

IF OBJECT_ID('dbo.View_StudentsSimple', 'V') IS NOT NULL
    DROP VIEW dbo.View_StudentsSimple;
GO

CREATE VIEW dbo.View_StudentsSimple
AS
SELECT
    IDSTUDENT AS [Код студента],
    NAME      AS [ФИО],
    IDGROUP   AS [Код группы],
    BDAY      AS [Дата рождения],
    STAMP     AS [Зачётка]
FROM STUDENT;
GO

**Результат:**

| Код студента | ФИО | Код группы | Дата рождения | Зачётка |
|---|---|---|---|---|
| 1 | Алексеев Алексей Алексеевич | 3 | 2005-03-12 | ST-001 |
| 2 | Борисова Борислава Ивановна | 1 | 2005-07-25 | ST-002 |
| 3 | Владимиров Владимир Петрович | 1 | 2005-01-18 | ST-003 |
| 4 | Григорьева Галина Сергеевна | 2 | 2004-11-05 | ST-004 |
| 5 | Дмитриев Дмитрий Олегович | 2 | 2004-09-14 | ST-005 |
| 6 | Егорова Екатерина Андреевна | 3 | 2005-05-30 | ST-006 |
| 7 | Жуков Ждан Викторович | 3 | 2005-08-21 | ST-007 |
| 8 | Зайцева Зоя Михайловна | 4 | 2003-02-11 | ST-008 |
| 9 | Ильин Игорь Станиславович | 5 | 2005-06-07 | ST-009 |

**Почему модифицируемое:**
- Одна базовая таблица (STUDENT).
- Нет агрегатов, DISTINCT, GROUP BY, HAVING, подзапросов.
- Можно делать INSERT/UPDATE/DELETE через представление.

---

### 2. Аналитическое представление View_PulpitLoad (3 таблицы)

IF OBJECT_ID('dbo.View_PulpitLoad', 'V') IS NOT NULL
    DROP VIEW dbo.View_PulpitLoad;
GO

CREATE VIEW dbo.View_PulpitLoad
AS
SELECT
    P.PULPIT_NAME               AS [Кафедра],
    F.FACULTY_NAME              AS [Факультет],
    COUNT(DISTINCT S.[SUBJECT]) AS [Дисциплин],
    COUNT(DISTINCT T.TEACHER)   AS [Преподавателей]
FROM PULPIT P
JOIN FACULTY F  ON F.FACULTY = P.FACULTY
LEFT JOIN SUBJECT S ON S.PULPIT = P.PULPIT
LEFT JOIN TEACHER T ON T.PULPIT = P.PULPIT
GROUP BY P.PULPIT_NAME, F.FACULTY_NAME;
GO

**Результат:**

| Кафедра | Факультет | Дисциплин | Преподавателей |
|---|---|---|---|
| Кафедра информатики и вычислительной техники | Факультет информационных технологий | 2 | 2 |
| Кафедра лингвистики | Факультет филологии | 1 | 1 |
| Кафедра математики | Физико-математический факультет | 2 | 2 |
| Кафедра менеджмента | Факультет экономики и управления | 1 | 1 |
| Кафедра правоведения | Юридический факультет | 1 | 1 |
| Кафедра программной инженерии | Факультет информационных технологий | 2 | 2 |
| Кафедра физики | Физико-математический факультет | 1 | 1 |
| Кафедра экономики | Факультет экономики и управления | 1 | 1 |

**Что показывает:** нагрузку по кафедрам — сколько дисциплин и преподавателей.

---

### 3. Аналитическое представление View_StudentProgress (4 таблицы)

IF OBJECT_ID('dbo.View_StudentProgress', 'V') IS NOT NULL
    DROP VIEW dbo.View_StudentProgress;
GO

CREATE VIEW dbo.View_StudentProgress
AS
SELECT
    F.FACULTY_NAME                                          AS [Факультет],
    G.COURSE                                                AS [Курс],
    S.NAME                                                  AS [Студент],
    COUNT(P.[SUBJECT])                                      AS [Оценок],
    CAST(AVG(CAST(P.NOTE AS DECIMAL(4,2))) AS DECIMAL(4,2)) AS [Средний балл],
    MAX(P.NOTE)                                             AS [Лучшая],
    MIN(P.NOTE)                                             AS [Худшая]
FROM STUDENT S
JOIN GROUPS   G ON G.IDGROUP   = S.IDGROUP
JOIN FACULTY  F ON F.FACULTY   = G.FACULTY
JOIN PROGRESS P ON P.IDSTUDENT = S.IDSTUDENT
WHERE P.NOTE >= 4
  AND G.COURSE BETWEEN 1 AND 4
GROUP BY F.FACULTY_NAME, G.COURSE, S.NAME
HAVING COUNT(P.[SUBJECT]) >= 1;
GO

**Результат:**

| Факультет | Курс | Студент | Оценок | Средний балл | Лучшая | Худшая |
|---|---|---|---|---|---|---|
| Факультет филологии | 1 | Орлова Ольга Витальевна | 1 | 10.00 | 10 | 10 |
| Факультет экономики и управления | 2 | Лебедев Леонид Артёмович | 1 | 9.00 | 9 | 9 |
| Физико-математический | 1 | Михайлова Марина Павловна | 1 | 9.00 | 9 | 9 |
| Факультет информационных технологий | 1 | Владимиров Владимир Петрович | 1 | 8.00 | 8 | 8 |
| Факультет информационных технологий | 2 | Григорьева Галина Сергеевна | 1 | 8.00 | 8 | 8 |
| … | … | … | … | … | … | … |

**Что показывает:** сводку успеваемости по студентам с фильтрами (оценка ≥ 4, курс 1–4).

---

## Проверка: список созданных представлений

SELECT name AS [Представление], create_date AS [Создано]
FROM sys.views
WHERE name LIKE 'View_%'
ORDER BY name;

| Представление | Создано |
|---|---|
| View_PulpitLoad | 2026-09-25 22:46:22.857 |
| View_StudentProgress | 2026-09-25 22:46:22.883 |
| View_StudentsSimple | 2026-09-25 22:46:22.850 |

---

## Проверка модифицируемости View_StudentsSimple

UPDATE dbo.View_StudentsSimple
SET [ФИО] = 'Иванов Иван Иванович'
WHERE [Код студента] = 1;
GO

SELECT IDSTUDENT, NAME FROM STUDENT WHERE IDSTUDENT = 1;

**Результат:**

| IDSTUDENT | NAME |
|---|---|
| 1 | Иванов Иван Иванович |

Изменение через представление **попало в базовую таблицу** — представление **модифицируемое**.

---

## Анализ

### Модифицируемое представление

View_StudentsSimple — **модифицируемое**:
- Одна базовая таблица STUDENT.
- Нет агрегатов, DISTINCT, GROUP BY, подзапросов.
- UPDATE через представление изменил данные в STUDENT.

### Аналитические представления

View_PulpitLoad и View_StudentProgress — **не модифицируемые**:
- Содержат агрегаты (COUNT, AVG, MAX, MIN).
- Содержат GROUP BY и HAVING.
- Через них **нельзя** делать INSERT/UPDATE/DELETE.

### Отличие от View_StudentsSimple

| Параметр | View_StudentsSimple | View_PulpitLoad | View_StudentProgress |
|---|---|---|---|
| Таблиц | 1 | 3 | 4 |
| Агрегаты | Нет | COUNT | COUNT, AVG, MAX, MIN |
| GROUP BY | Нет | Да | Да |

---

## Вывод

Созданы **три представления**:

1. **View_StudentsSimple** — **модифицируемое**, одна таблица STUDENT. Через него можно делать INSERT/UPDATE/DELETE.
2. **View_PulpitLoad** — **аналитическое**, 3 таблицы, GROUP BY. Показывает нагрузку по кафедрам.
3. **View_StudentProgress** — **аналитическое**, 4 таблицы, GROUP BY + фильтры. Показывает успеваемость студентов.

**Модифицируемость проверена** через UPDATE и последующую проверку базовой таблицы STUDENT. Изменения через представление корректно попали в базовую таблицу.

Представления упрощают доступ к сложным данным и позволяют использовать одни и те же запросы многократно.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база college.