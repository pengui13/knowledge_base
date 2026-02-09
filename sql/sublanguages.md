# SQL (Structured Query Language)

## SQL vs NoSQL

Порівняння:

Структура:
SQL: Таблиці
NoSQL: Документи, ключ-значення, графи

Дані Схема:
SQL: Фіксована
NoSQL: Гнучка

Мова:
SQL: SQL
NoSQL: Різна

ACID:
SQL: Так
NoSQL: Часто ні

Масштабування:
SQL: Вертикальне
NoSQL: Горизонтальне

Приклади:
SQL: MySQL, PostgreSQL
NoSQL: MongoDB, Redis, Neo4j

## Підвиди SQL

Підвиди - Назва:

Підмова - Для чого:
DDL - Data Definition Language - Структура (таблиці, індекси)
DML - Data Manipulation Language - Дані (вставити, читати, змінити)
DCL - Data Control Language - Права доступу
TCL - Transaction Control Language - Транзакції
DQL - Data Query Language - Для сприйняття

## DDL (Data Definition Language)

CREATE - Створити
ALTER - Змінити
DROP - Видалити
TRUNCATE - Очистити

### CREATE

Приклади:

CREATE DATABASE university;

CREATE TABLE students (
  student_id INT PRIMARY KEY AUTO_INCREMENT,
  gpa DECIMAL(3,2)
);

## DML (Data Manipulation Language)

Робота з даними:

SELECT - Читати
INSERT - Вставити
UPDATE - Оновити
DELETE - Видалити (наділове приклади не табл.)

### SELECT

SELECT * FROM students WHERE age = 20;

### INSERT

INSERT INTO students (name, age) VALUES ('John', 20);

### UPDATE

UPDATE students SET age = 21 WHERE student_id = 1;

### DELETE

DELETE FROM students WHERE age < 18;

## DCL (Data Control Language)

Керування правами доступу:

GRANT - Надати
REVOKE - Забрати (право на SELECT)

### Приклади GRANT

GRANT SELECT ON students TO user1;
- Кінця право

GRANT SELECT, INSERT, UPDATE ON students to user1;
- Всі права на всю базу

GRANT ALL PRIVILEGES ON students TO user1;
- Право створок, користувачів

GRANT CREATE USER TO admin;

### Приклади REVOKE

- З правом передавати іншим

GRANT SELECT ON students TO user1 WITH GRANT OPTION;

REVOKE:

REVOKE INSERT ON students FROM user1;
REVOKE ALL PRIVILEGES on students FROM user1;

## DQL (Data Query Language)

LIMIT (по кількості, сортує і залишає/ський порядок) - відпали n рядків
ORDER BY - сортує
DISTINCT - Залишає 3 дублікати
HAVING - Фільтрує результат GROUP BY
GROUP BY - Попу рядки, що мають однак, знач. в запаленому стовпці
WHERE - Фільтрує дані перед групуванням/агрегацією
FROM - Початує табл. з якої брати дані

## TCL (Transaction Control Language)

Команди - Що робить:

BEGIN / START TRANSACTION - Почати транз.
COMMIT - Зберегти зміни
ROLLBACK - Відміни зміни
SAVEPOINT - Точка збереження

### Приклад

START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE acc_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE acc_id = 2;
COMMIT; (якщо все гід)

Інакше:
ROLLBACK;

### SAVEPOINT

START TRANSACTION;
INSERT INTO ...
SAVEPOINT order_created;
INSERT ... order_items
INSERT ...
(Щось пішло не так з items)
ROLLBACK TO order_created;
COMMIT;

## ALTER

ALTER TABLE students ADD phone VARCHAR(20);
ALTER TABLE students MODIFY phone VARCHAR(50);
ALTER TABLE students DROP COLUMN phone;
ALTER TABLE students RENAME COLUMN name to fullname;
ALTER TABLE students ADD CONSTRAINT chk_age CHECK (gpa >= 0 AND gpa <= 4);

## DROP

DROP TABLE students;
DROP TABLE IF EXISTS students;
DROP d DATABASE university;
DROP INDEX idx_name ON students;

## TRUNCATE

TRUNCATE TABLE students; (Видалити всі дані, залишити структуру)

### TRUNCATE vs DELETE

TRUNCATE - DELETE:

Що видаляє: Всі рядки - Можна з WHERE
Швидкість: Швидко - Повільніше
Лог: Мінімальний - Повніша
Відкат: Не можна - Можна
AUTO_INCREMENT: Скидає - Не скидає
