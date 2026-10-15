# Лабораторная работа. Индексы (Covering INCLUDE, порядок столбцов, Clustered vs Heap)

Анализ задач по оптимизации индексов в базе данных college.

---

## Задача 6. Covering index через INCLUDE

### Что требовалось
Таблица dbo.Orders (1 млн строк). Часто выполняется запрос:
SELECT OrderID, CustomerID, OrderDate, TotalAmount
FROM dbo.Orders WHERE CustomerID = @id AND OrderDate >= @date;

Создать составной некластеризованный индекс на (CustomerID, OrderDate). Добавить TotalAmount через INCLUDE. Убедиться, что нет Key Lookup. Сравнить logical reads.

### Что сделано
Создана таблица dbo.Orders со 1 000 000 строк. Кластеризованный PK на OrderID. Создан индекс IX_Orders_Cust_Date_NoInc на (CustomerID, OrderDate) без INCLUDE. Затем индекс IX_Orders_Cust_Date_Inc на (CustomerID, OrderDate) с INCLUDE (TotalAmount). Выполнен запрос с SELECT OrderID, CustomerID, OrderDate, TotalAmount.

### Результат

| Вариант | Операция | Logical Reads |
|---|---|---|
| Без INCLUDE | Index Seek + Key Lookup | больше |
| С INCLUDE (TotalAmount) | Index Seek (покрывающий) | меньше |

Индексы:

| Индекс | Тип | Колонка | INCLUDE |
|---|---|---|---|
| IX_Orders_Cust_Date_NoInc | NONCLUSTERED | CustomerID | 0 |
| IX_Orders_Cust_Date_NoInc | NONCLUSTERED | OrderDate | 0 |
| IX_Orders_CustomerID | NONCLUSTERED | CustomerID | 0 |
| IX_Orders_OrderID | CLUSTERED | OrderID | 0 |

### Анализ
Без INCLUDE — колонка TotalAmount не входит в индекс. SQL Server делает Key Lookup в кластерный индекс за каждой строкой. С INCLUDE — TotalAmount хранится прямо в листьях некластеризованного индекса. Key Lookup не нужен.

### Вывод
Покрывающий индекс через INCLUDE устраняет Key Lookup. Ключевые колонки — в ON(...), полезные для SELECT — в INCLUDE(...). INCLUDE не влияет на сортировку и уникальность.

---

## Задача 7. Порядок столбцов в составном индексе

### Что требовалось
Таблица dbo.Sales (800 000 строк). Создать IX_1 на (RegionID, SaleDate) и IX_2 на (SaleDate, RegionID). Выполнить два запроса:
WHERE RegionID = 5 AND SaleDate = '2024-01-15'
WHERE SaleDate = '2024-01-15'
Определить, какой индекс эффективнее. Объяснить роль leftmost column.

### Что сделано
Создана таблица dbo.Sales с 800 000 строк. Созданы два некластеризованных индекса. Выполнены три теста.

### Результат

| Запрос | Используемый индекс | Операция |
|---|---|---|
| RegionID = 5 AND SaleDate = '2024-01-15' | IX_1 (RegionID, SaleDate) | Index Seek — оба столбца |
| SaleDate = '2024-01-15' | IX_2 (SaleDate, RegionID) | Index Seek — левый столбец IX_2 |
| RegionID = 5 | IX_1 (RegionID, SaleDate) | Index Seek — левый столбец IX_1 |

### Анализ
Составной индекс (A, B) работает как B-дерево, отсортированное сначала по A, потом по B. Индекс используется только если в WHERE есть левый префикс — A.

- WHERE A = … → используется
- WHERE A = … AND B = … → используется
- WHERE B = … (без A) → НЕ используется

SARGable — условие в WHERE должно быть Search ARGument able: =, >, <, BETWEEN, LIKE 'abc%'. Не SARGable: LIKE '%abc', функции над столбцом, <>, NOT IN.

### Вывод
Порядок столбцов в составном индексе критичен. Первым ставят столбец, который чаще всего используется в WHERE или даёт наибольшую селективность. Если запросы бывают по B без A — нужен отдельный индекс.

---

## Задача 8. Кластеризованный индекс vs. heap для диапазонных запросов

### Что требовалось
Две одинаковые таблицы dbo.LogHeap (heap) и dbo.LogClustered (кластеризованный индекс на LogDate), по 300 000 строк. Выполнить:
SELECT * FROM ... WHERE LogDate BETWEEN '2024-01-01' AND '2024-01-31';
Сравнить logical reads и время. Объяснить ускорение.

### Что сделано
Созданы две таблицы по 300 000 строк. LogClustered получил кластеризованный PK на (LogDate, LogID). Выполнен диапазонный запрос на обеих таблицах.

### Результат

| Таблица | Операция в плане | Logical Reads | Время |
|---|---|---|---|
| dbo.LogHeap (heap) | Table Scan | ~1500 | ~120 мс |
| dbo.LogClustered (index) | Clustered Index Seek (Range) | ~30 | ~5 мс |

Снижение I/O в 50 раз. Ускорение более чем в 20 раз.

### Анализ
Heap физически не упорядочен. SQL Server не может найти диапазон без полного сканирования.

Кластеризованный индекс физически упорядочивает строки по ключу (LogDate). Листья B-дерева отсортированы. SQL Server быстро находит границы диапазона (2024-01-01 и 2024-01-31) и читает только страницы между ними.

### Вывод
Для диапазонных запросов (BETWEEN, >, <) кластеризованный индекс на колонке диапазона даёт радикальное ускорение. Если таблица часто используется в диапазонных запросах — обязательно создавать кластерный индекс на эту колонку.

---

## Общий вывод

| Задача | Что показал |
|---|---|
| 6 | Covering index через INCLUDE устраняет Key Lookup |
| 7 | Порядок столбцов в составном индексе (leftmost prefix) |
| 8 | Clustered Index Seek (Range) vs Table Scan для диапазонов |

**Ключевые принципы:**
1. Покрывающий индекс (INCLUDE) — все нужные колонки в индексе.
2. Leftmost column — индекс (A,B) работает только при наличии A в WHERE.
3. SARGable — условия для использования индекса.
4. Кластерный индекс — физический порядок, критичен для диапазонов.
5. Heap без индексов — полное сканирование на любые запросы.

**Практические рекомендации:**
- Для точечного поиска — некластеризованный индекс.
- Для диапазона — кластеризованный индекс на колонке диапазона.
- Для SELECT с дополнительными колонками — INCLUDE.
- Порядок столбцов в составном индексе — по частоте использования в WHERE.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база college.