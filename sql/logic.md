# Реляційна алгебра

Математична основа SQL. Набір операцій над таблицями.

## Операції

| Операція | Символ | Що робить |
|---|---|---|
| Вибір (Selection) | σ | Фільтрує рядки |
| Проєкція (Projection) | π | Вибирає стовпці |
| Об'єднання (Union) | ∪ | Зливає таблиці |
| Перетин (Intersection) | ∩ | Спільні рядки |
| Різниця (Difference) | − | Рядки А, яких немає в В |
| Декартовий добуток | × | Всі комбінації рядків |
| З'єднання (JOIN) | ⋈ | Комбінує по умові |
| Ділення (Division) | ÷ | Знаходить "всі" |

---

## 1. Вибір — σ

Фільтрує рядки за умовою.

```
Алгебра: σ(Age = 20)(Students)
```
```sql
SELECT * FROM students WHERE age = 20;
```

---

## 2. Проєкція — π

Вибирає певні стовпці. **Автоматично видаляє дублікати** — на відміну від SQL, де потрібен `DISTINCT`.

```
Алгебра: π(Name, Age)(Students)
```
```sql
SELECT DISTINCT Name, Age FROM Students;
```

---

## 3. Об'єднання — ∪

Зливає рядки з двох таблиць, видаляє дублікати. Таблиці повинні мати однакову структуру.

```
Алгебра: Students_A ∪ Students_B
```
```sql
-- без дублікатів
SELECT * FROM Students_A
UNION
SELECT * FROM Students_B;

-- з дублікатами
SELECT * FROM Students_A
UNION ALL
SELECT * FROM Students_B;
```

---

## 4. Перетин — ∩

Повертає тільки спільні рядки з обох таблиць.

```
Алгебра: Students_A ∩ Students_B
```
```sql
SELECT * FROM Students_A
INTERSECT
SELECT * FROM Students_B;

-- альтернатива (якщо INTERSECT не підтримується)
SELECT * FROM Students_A
WHERE ID IN (SELECT ID FROM Students_B);
```

---

## 5. Різниця — −

Повертає рядки з A, яких немає в B.

```
Алгебра: Students_A − Students_B
```
```sql
SELECT * FROM Students_A
EXCEPT
SELECT * FROM Students_B;

-- альтернатива
SELECT * FROM Students_A
WHERE ID NOT IN (SELECT ID FROM Students_B);
```

---

## 6. Декартовий добуток — ×

Всі можливі комбінації рядків двох таблиць. Якщо A має 3 рядки, B має 4 — результат 12 рядків.

```
Алгебра: Students × Courses
```
```sql
SELECT * FROM Students CROSS JOIN Courses;
-- або
SELECT * FROM Students, Courses;
```

---

## 7. З'єднання — ⋈

Декартовий добуток + фільтр по умові. Основа всіх JOIN.

```
Алгебра: Students ⋈(Students.ID = Enrollments.Student_ID) Enrollments
```
```sql
SELECT s.Name, c.Title
FROM Students s
INNER JOIN Enrollments e ON s.ID = e.Student_ID
INNER JOIN Courses c ON e.Course_ID = c.ID;
```

### Типи JOIN

| Тип | Що повертає |
|---|---|
| `INNER JOIN` | Тільки збіги з обох таблиць |
| `LEFT JOIN` | Всі з лівої + збіги з правої |
| `RIGHT JOIN` | Всі з правої + збіги з лівої |
| `FULL JOIN` | Всі рядки з обох таблиць |
| `CROSS JOIN` | Декартовий добуток |
| `NATURAL JOIN` | Автоматично з'єднує по однакових колонках |

---

## 8. Ділення — ÷

Найскладніша операція. Відповідає на питання **"Хто має ВСЕ?"**

**Питання:** Які студенти записалися на ВСІ курси?

```
Алгебра: Enrollments ÷ Courses
```

Дані:

| Student_ID | Course_ID |
|---|---|
| 1 | 1 |
| 1 | 2 |
| 2 | 1 |

Студент 1 записаний на курси 1 і 2 — на всі. Студент 2 тільки на курс 1 — не на всі.

```sql
SELECT Student_ID
FROM Enrollments
GROUP BY Student_ID
HAVING COUNT(DISTINCT Course_ID) = (SELECT COUNT(*) FROM Courses);
```
