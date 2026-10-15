# Лабораторная работа. Создание базы данных BookStoreDB

Анализ создания БД BookStoreDB: 5 таблиц, связи, тестовые данные, диаграмма.

---

## Что требовалось
Создать базу данных BookStoreDB. В ней — минимум 5 таблиц, связанных между собой, кроме одной изолированной. Наполнить тестовыми данными. Выбрать по одной строке из каждой таблицы. Построить диаграмму БД.

---

## Что сделано

### 1. Создание базы данных

CREATE DATABASE BookStoreDB;
GO
USE BookStoreDB;
GO

Скриншот: окно «Создание базы данных» в SSMS, имя BookStoreDB.

### 2. Создание 5 таблиц

**Таблица 1: Authors** — авторы (связанная).

CREATE TABLE Authors (
    AuthorID INT PRIMARY KEY IDENTITY(1,1),
    FullName NVARCHAR(100) NOT NULL,
    Country NVARCHAR(50)
);

**Таблица 2: Books** — книги (связана с Authors).

CREATE TABLE Books (
    BookID INT PRIMARY KEY IDENTITY(1,1),
    Title NVARCHAR(150) NOT NULL,
    Price DECIMAL(10, 2) NOT NULL,
    AuthorID INT FOREIGN KEY REFERENCES Authors(AuthorID)
);

**Таблица 3: Orders** — заказы (связанная).

CREATE TABLE Orders (
    OrderID INT PRIMARY KEY IDENTITY(1,1),
    OrderDate DATETIME DEFAULT GETDATE(),
    CustomerName NVARCHAR(100) NOT NULL
);

**Таблица 4: OrderDetails** — детали заказа (связывает Books и Orders).

CREATE TABLE OrderDetails (
    OrderDetailID INT PRIMARY KEY IDENTITY(1,1),
    OrderID INT FOREIGN KEY REFERENCES Orders(OrderID),
    BookID INT FOREIGN KEY REFERENCES Books(BookID),
    Quantity INT NOT NULL
);

**Таблица 5: Promocodes** — промокоды (изолированная, БЕЗ связей).

CREATE TABLE Promocodes (
    PromoID INT PRIMARY KEY IDENTITY(1,1),
    Code NVARCHAR(20) UNIQUE NOT NULL,
    DiscountPercent INT NOT NULL,
    IsActive BIT DEFAULT 1
);

### 3. Наполнение тестовыми данными

INSERT INTO Authors (FullName, Country) VALUES
(N'Джордж Оруэлл', N'Великобритания'),
(N'Стивен Кинг', N'США');

INSERT INTO Books (Title, Price, AuthorID) VALUES
(N'1984', 550.00, 1),
(N'Сияние', 700.00, 2);

INSERT INTO Orders (CustomerName) VALUES
(N'Иван Иванов'),
(N'Анна Петрова');

INSERT INTO OrderDetails (OrderID, BookID, Quantity) VALUES
(1, 1, 1),
(2, 2, 2);

INSERT INTO Promocodes (Code, DiscountPercent) VALUES
(N'READ2026', 15),
(N'BOOKLOVE', 10);

### 4. Выборка по одной строке из каждой таблицы

SELECT TOP 1 * FROM Authors;
SELECT TOP 1 * FROM Books;
SELECT TOP 1 * FROM Orders;
SELECT TOP 1 * FROM OrderDetails;
SELECT TOP 1 * FROM Promocodes;

---

## Результат

**Authors:**

AuthorID | FullName | Country
1 | Джордж Оруэлл | Великобритания

**Books:**

BookID | Title | Price | AuthorID
1 | 1984 | 550.00 | 1

**Orders:**

OrderID | OrderDate | CustomerName
1 | 2026-09-17 21:46:20.993 | Иван Иванов

**OrderDetails:**

OrderDetailID | OrderID | BookID | Quantity
1 | 1 | 1 | 1

**Promocodes:**

PromoID | Code | DiscountPercent | IsActive
1 | READ2026 | 15 | 1

---

## Схема связей
