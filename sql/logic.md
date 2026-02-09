# Реляційна алгебра - мат. основа SQL. Набір операцій

## Над таблицями

Операція - Символ - Що робить:

Вибір (Selection) - σ - Фільтрує рядки
Проєкція (Projection) - π - Вибирає стовпці
Об'єднання (Union) - ∪ - Зливає таблиці
Перетин (Intersection) - ∩ - Спільні рядки
Різниця (Difference) - − - Рядки А, яких немає в В
Декартовий добуток - × - Всі комбінації рядків
З'єднання (JOIN) - ⋈ - Комбінує по умові
Ділення (Division) - ÷ - Знаходить "всі"

## 1. Вибір - σ

Алгебра: σ (Age = 20) (Students)

SQL: SELECT * FROM students WHERE age = 20;

## 2. Проєкція - π

Вибирає певні стовпці.

Алгебра: π (Name, Age) (Students)

SQL: SELECT Name, Age FROM Students;

### Проєкція - Видалює дублікати - Є SQL тільки DISTINCT

## 3. Об'єднання (UNION) - ∪

Зливає рядки з 2 табл., видаляє дублікати.

Алгебра: Students_A ∪ Students_B (щоби мати однакову структуру)

SQL: SELECT * FROM Students_A
     UNION
     SELECT * FROM Students_B

UNION ALL - Залишає дублікати:

SELECT * FROM Students_A
UNION ALL
SELECT * FROM Students_B

## 4. Перетин - ∩

Алгебра: Students_A ∩ Students_B

SQL: SELECT * FROM Students_A
     INTERSECT
     SELECT * FROM Students_B

Альтернатива (якщо INTERSECT не підтримується):

SELECT * FROM Students_A WHERE ID IN (SELECT ID FROM Students_B);

## 5. Різниця - −

Алгебра: Students_A − Students_B

SQL: SELECT * FROM Students_A
     EXCEPT
     SELECT * FROM Students_B;

Альтернатива:

SELECT * FROM Students_A
WHERE ID NOT IN (SELECT ID FROM Students_B);

## 6. Декартовий добуток (Cartesian Product) - ×

Алгебра: Students × Courses

SQL: SELECT * FROM Students, Courses;

або:

SELECT * FROM Students CROSS JOIN Courses;

## 7. З'єднання JOIN - ⋈

Декартовий добуток + фільтр по умові.

Алгебра: Students ⋈ (Students.ID = Enrollments.Student_ID) Enrollments

SQL:

SELECT s.Name, c.Title
FROM Students s
INNER JOIN Enrollments e ON s.ID = e.Student_ID
INNER JOIN Courses c ON e.Course_ID = c.ID

## Типи JOIN

### INNER JOIN - Тільки ЗПА

Всі з лівої + збіги з правої

### LEFT JOIN - Всі з лівої + збіги з правої

### RIGHT JOIN - Всі з правої + збіги з лівої

### FULL JOIN - Всі з обох

### NATURAL JOIN - Автоматично з'єднує по однакових колонках

## Ділення (Division) - ÷

Найскладніша операція. Знаходить "Ти, Хто має Все".

Питання: Які студенти записалися на ВСІ курси?

Enrollments:
Student_ID | Course_ID | ID
1          | 1         | 1
1          | 2         | 2
2          | 1         | Student_ID

Алгебра: Enrollments ÷ Courses

SQL (Через правого оператора):

Спосіб:

SELECT Student_ID
FROM Enrollments
GROUP BY Student_ID
HAVING COUNT(DISTINCT Course_ID) = (SELECT COUNT(*) FROM Courses);
