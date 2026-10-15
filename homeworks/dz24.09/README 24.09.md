# Лабораторная работа. Группировка и представления (VIEW)

Анализ способов группировки данных и создания представлений в базе college.

---

## Часть 1. Способы группировки

### 1. GROUP BY — обычная группировка

SELECT
    G.COURSE                    AS [Курс],
    COUNT(DISTINCT S.IDSTUDENT) AS [Студентов],
    COUNT(P.[SUBJECT])          AS [Оценок],
    CAST(AVG(CAST(P.NOTE AS DECIMAL(4,2))) AS DECIMAL(4,2)) AS [Средний балл]
FROM GROUPS G
LEFT JOIN STUDENT  S ON S.IDGROUP   = G.IDGROUP
LEFT JOIN PROGRESS P ON P.IDSTUDENT = S.IDSTUDENT
GROUP BY G.COURSE
ORDER BY G.COURSE;

Результат:

| Курс | Студентов | Оценок | Средний балл |
|---|---|---|---|
| 1 | 11 | 10 | 7.50 |
| 2 | 4 | 5 | 7.80 |
| 3 | 2 | 1 | 8.00 |

---

### 2. ROLLUP — иерархические итоги

SELECT
    G.FACULTY AS [Факультет],
    G.COURSE AS [Курс],
    COUNT(DISTINCT S.IDSTUDENT) AS [Студентов]
FROM GROUPS G
LEFT JOIN STUDENT S ON S.IDGROUP = G.IDGROUP
GROUP BY ROLLUP (G.FACULTY, G.COURSE)
ORDER BY G.FACULTY, G.COURSE;

Результат (15 строк):

| Факультет | Курс | Студентов |
|---|---|---|
| NULL | NULL | 17 |
| FEN | NULL | 3 |
| FEN | 1 | 2 |
| FEN | 2 | 1 |
| FFL | NULL | 1 |
| FIT | NULL | 10 |
| … | … | … |

NULL в колонке — признак итоговой строки.

---

### 3. CUBE — все комбинации

SELECT
    G.FACULTY AS [Факультет],
    G.COURSE AS [Курс],
    COUNT(DISTINCT S.IDSTUDENT) AS [Студентов]
FROM GROUPS G
LEFT JOIN STUDENT S ON S.IDGROUP = G.IDGROUP
GROUP BY CUBE (G.FACULTY, G.COURSE)
ORDER BY G.FACULTY, G.COURSE;

Результат (18 строк) — добавляет итоги по курсу без факультета:

| Факультет | Курс | Студентов |
|---|---|---|
| NULL | NULL | 17 |
| NULL | 1 | 11 |
| NULL | 2 | 4 |
| NULL | 3 | 2 |
| FEN | NULL | 3 |
| FEN | 1 | 2 |
| … | … | … |

---

### 4. GROUPING SETS — выборочные итоги

Позволяет явно указать нужные наборы группировок, исключив ненужные итоги.

---

## Таблица сравнения способов группировки

| Способ | Какие строки даёт | Кол-во строк | NULL-строк | Когда использовать |
|---|---|---|---|---|
| GROUP BY | Только детализация по всем колонкам | Минимум | Нет | Обычные отчёты |
| ROLLUP(A, B) | (A,B), (A), () | Больше | Да | Иерархия: от детализации к общему итогу |
| CUBE(A, B) | (A,B), (A), (B), () | Максимум | Да | Многомерный анализ — нужны все срезы |
| GROUPING SETS | Только указанные наборы | Ровно сколько задал | Да (если нужен) | Выборочные итоги — контроль над результатом |

---

## Главные отличия

1. **GROUP BY** — база. Никаких итогов.

2. **ROLLUP** — иерархия. Итоги «снизу вверх»: сначала детализация, потом частичные итоги по первому столбцу, потом общий.

3. **CUBE** — то же, что ROLLUP, но добавляет итоги по второму столбцу без первого. Например, ROLLUP(Факультет, Курс) даёт итог по факультету, а CUBE(Факультет, Курс) — ещё и итог по курсу без факультета.

4. **GROUPING SETS** — самый гибкий. Ты сам перечисляешь, какие итоги нужны. Не хочешь общий итог — не указывай ().

---

## Часть 2. Создание представления для своей БД

### Требования
- Не менее 2 таблиц.
- Для столбцов псевдонимы на русском языке.
- Использовать группировку и/или несколько фильтров.
- Сортировка.

### Реализация

CREATE VIEW dbo.View_StudentProgress
AS
SELECT
    F.FACULTY_NAME                              AS [Факультет],
    G.COURSE                                    AS [Курс],
    S.NAME                                      AS [Студент],
    COUNT(P.[SUBJECT])                          AS [Количество оценок],
    CAST(AVG(CAST(P.NOTE AS DECIMAL(4,2))) AS DECIMAL(4,2)) AS [Средний балл],
    MAX(P.NOTE)                                 AS [Лучшая оценка],
    MIN(P.NOTE)                                 AS [Худшая оценка]
FROM STUDENT S
JOIN GROUPS   G ON G.IDGROUP   = S.IDGROUP
JOIN FACULTY  F ON F.FACULTY   = G.FACULTY
JOIN PROGRESS P ON P.IDSTUDENT = S.IDSTUDENT
WHERE P.NOTE >= 4
  AND G.COURSE BETWEEN 1 AND 4
GROUP BY F.FACULTY_NAME, G.COURSE, S.NAME
HAVING COUNT(P.[SUBJECT]) >= 1;
GO

### Проверка по требованиям

| Требование | Реализация |
|---|---|
| ≥ 2 таблиц | 4 таблицы: STUDENT, GROUPS, FACULTY, PROGRESS |
| Псевдонимы на русском | [Факультет], [Курс], [Студент], [Средний балл] |
| Группировка | GROUP BY + HAVING |
| Фильтры | WHERE P.NOTE >= 4 AND G.COURSE BETWEEN 1 AND 4 |
| Сортировка | (в SELECT через ORDER BY при вызове) |

### Запрос к представлению

SELECT * FROM dbo.View_StudentProgress
ORDER BY [Средний балл] DESC, [Студент];
GO

---

## Общий вывод

| Способ | Что показал |
|---|---|
| GROUP BY | Базовая группировка без итогов |
| ROLLUP | Иерархические итоги (факультет → курс → общий) |
| CUBE | Все возможные срезы |
| GROUPING SETS | Точный контроль над итогами |
| VIEW | Абстракция сложного запроса |

**Ключевые принципы:**
1. GROUP BY — база для агрегатов.
2. ROLLUP — для иерархий.
3. CUBE — для многомерного анализа.
4. GROUPING SETS — для выборочных итогов.
5. VIEW — сохранённый запрос для повторного использования.

**Практическая ценность:**
- Представления упрощают доступ к сложным данным.
- Модифицируемые представления позволяют менять данные через них.
- Группировка с ROLLUP/CUBE/GROUPING SETS — основа аналитических отчётов.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база college.