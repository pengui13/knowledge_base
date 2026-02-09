# Основні концепції баз даних

## Моделі організації даних

| Модель | Опис | Приклади |
|--------|------|----------|
| **Реляційна** | Таблиці, зв'язки через ключі | MySQL, PostgreSQL |
| **Документна** | JSON/BSON документи | MongoDB, CouchDB |
| **Ключ-значення** | Пара key:value | Redis, Memcached |
| **Графова** | Вузли та зв'язки між ними | Neo4j, Amazon Neptune |

---

## SQL (Structured Query Language)

Декларативна мова для виконання операцій над даними в реляційних базах.

**Категорії команд:**
- **DDL** (Data Definition): `CREATE`, `ALTER`, `DROP`
- **DML** (Data Manipulation): `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- **DCL** (Data Control): `GRANT`, `REVOKE`
- **TCL** (Transaction Control): `BEGIN`, `COMMIT`, `ROLLBACK`

---

## Транзакція

Група операцій, які виконуються як одне ціле — атомарно.

**Приклад банківського переказу:**
```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;
```

**Результат:** або обидві операції виконані успішно, або жодна (rollback).

---

## ACID властивості транзакцій

| Властивість | Опис | Приклад |
|-------------|------|---------|
| **A — Atomicity** | Всі операції виконуються повністю або жодна | Переказ: або списали і зарахували, або нічого |
| **C — Consistency** | Дані залишаються валідними до і після транзакції | Сума грошей у системі не змінюється |
| **I — Isolation** | Транзакції не бачать незавершені зміни одна одної | Паралельні транзакції працюють незалежно |
| **D — Durability** | Після COMMIT зміни зберігаються назавжди | Навіть при збої сервера дані не втрачаються |

---

## Індекси

Структури даних для прискорення пошуку в таблицях.

### Порівняння: без індексу vs з індексом

```sql
-- Без індексу: Full Table Scan
SELECT * FROM users WHERE email = 'test@gmail.com';
-- ⏱ ~2 секунди (перебір 1 млн рядків)

-- Створення індексу
CREATE INDEX idx_email ON users(email);

-- З індексом: пошук через B-Tree
SELECT * FROM users WHERE email = 'test@gmail.com';
-- ⏱ ~0.001 секунди
```

---

### Типи індексів

#### 1. B-Tree (Balanced Tree)

Збалансоване дерево — стандартний і найпоширеніший тип індексу.

**Структура:**
```
           [50]
          /    \
       [25]    [75]
       /  \    /  \
    [10] [30] [60] [90]
```

**Принцип роботи:**
- Пошук від кореня до листка (O(log n))
- Дерево автоматично балансується при вставках

**Підтримує операції:**
| Операція | Приклад |
|----------|---------|
| Рівність | `WHERE id = 5` |
| Порівняння | `WHERE age > 18` |
| Діапазон | `WHERE age BETWEEN 18 AND 25` |
| Сортування | `ORDER BY created_at` |
| Префікс | `WHERE name LIKE 'Alex%'` |

**Приклад:**
```sql
CREATE INDEX idx_age ON users(age);
SELECT * FROM users WHERE age BETWEEN 18 AND 25;
-- B-Tree швидко знаходить діапазон значень
```

---

#### 2. Hash індекс

Хеш-таблиця для надшвидкого пошуку за точним значенням.

**Принцип роботи:**
```
email = "test@gmail.com"
         ↓
      hash()
         ↓
    bucket #7 → рядок #17
```

| Підтримує | НЕ підтримує |
|-----------|--------------|
| `=` | `>`, `<`, `>=`, `<=` |
| | `BETWEEN` |
| | `LIKE` |
| | `ORDER BY` |

**Ідеально для:**
- Session ID
- UUID / токени
- Lookup таблиці

---

#### 3. Full-Text індекс

Спеціалізований індекс для пошуку слів у текстових полях.

**Чому B-Tree не підходить:**
```sql
-- LIKE '%database%' — повний перебір, індекс ігнорується
SELECT * FROM articles WHERE content LIKE '%database%';
```

**Full-Text рішення:**
```sql
CREATE FULLTEXT INDEX idx_content ON articles(content);
SELECT * FROM articles 
WHERE MATCH(content) AGAINST('database technology');
```

**Переваги:**
- Швидкий пошук слів у великих текстах
- Ігнорує стоп-слова ("the", "a", "is")
- Розуміє морфологію (run → running, ran)
- Ранжування за релевантністю

---

### Коли використовувати який індекс

| Сценарій | Тип індексу |
|----------|-------------|
| Універсальний пошук + діапазони + сортування | **B-Tree** |
| Тільки точний пошук (session, UUID) | **Hash** |
| Пошук слів у тексті (статті, коментарі) | **Full-Text** |
| Геолокаційні дані | **Spatial (R-Tree)** |

---

## Backup стратегії

### Типи резервного копіювання

| Тип | Що копіюємо | Розмір | Відновлення |
|-----|-------------|--------|-------------|
| **Full** | Всю БД повністю | 100 ГБ | Тільки цей backup |
| **Incremental** | Зміни з останнього backup | ~50 МБ/день | Full + всі incremental |
| **Differential** | Зміни з останнього FULL | Зростає щодня | Full + останній diff |

**Приклад за тиждень:**

```
Неділя: Full Backup (100 ГБ)
        │
        ├── Incremental стратегія:
        │   Пн: +50 МБ (зміни з Нд)
        │   Вт: +30 МБ (зміни з Пн)
        │   Ср: +40 МБ (зміни з Вт)
        │   Відновлення: Full + Пн + Вт + Ср
        │
        └── Differential стратегія:
            Пн: +50 МБ  (зміни з Нд)
            Вт: +80 МБ  (зміни з Нд)
            Ср: +120 МБ (зміни з Нд)
            Відновлення: Full + Ср
```

**Компроміси:**
- Incremental: менше місця, складніше відновлення
- Differential: більше місця, простіше відновлення

---

## Розподілені бази даних

### Навіщо розподіляти?

1. **Масштабування** — більше даних, ніж вміщує один сервер
2. **Відмовостійкість** — один сервер впав, інші працюють
3. **Латентність** — сервер ближче до користувача

---

### Шардінг (Sharding)

Горизонтальний розподіл даних між серверами.

```
┌─────────────────┐    ┌─────────────────┐
│   Shard 1       │    │   Shard 2       │
│   Users A-M     │    │   Users N-Z     │
│   500K records  │    │   500K records  │
└─────────────────┘    └─────────────────┘
```

**Стратегії шардінгу:**
| Стратегія | Опис | Плюси/Мінуси |
|-----------|------|--------------|
| **Range** | За діапазоном (A-M, N-Z) | Простий, але hotspots |
| **Hash** | `hash(user_id) % N` | Рівномірний розподіл |
| **Directory** | Lookup таблиця | Гнучкий, але bottleneck |

---

### Реплікація (Replication)

Копіювання даних на кілька серверів.

**Master-Slave (Primary-Replica):**
```
       ┌──────────┐
       │  Master  │ ←── Всі WRITE
       │   (RW)   │
       └────┬─────┘
            │ async/sync реплікація
      ┌─────┴─────┐
      ↓           ↓
┌──────────┐ ┌──────────┐
│  Slave   │ │  Slave   │ ←── READ
│   (R)    │ │   (R)    │
└──────────┘ └──────────┘
```

**Master-Master:**
```
┌──────────┐ ←─────────→ ┌──────────┐
│  Master  │    sync     │  Master  │
│   (RW)   │             │   (RW)   │
└──────────┘             └──────────┘
```

⚠️ **Проблема:** конфлікти при одночасному записі

---

## OLTP vs OLAP

### Порівняння

| Характеристика | OLTP | OLAP |
|----------------|------|------|
| **Призначення** | Операційні транзакції | Аналітика та звіти |
| **Операції** | INSERT, UPDATE, DELETE | SELECT (складні запити) |
| **Запити** | Прості, швидкі | Складні, з агрегаціями |
| **Час відповіді** | Мілісекунди | Секунди — хвилини |
| **Нормалізація** | 3NF (нормалізовано) | Денормалізовано (Star/Snowflake) |
| **Дані** | Поточні, актуальні | Історичні, агреговані |
| **Користувачі** | Тисячі (оператори, клієнти) | Десятки (аналітики, менеджери) |

---

### OLTP — Online Transaction Processing

**Характеристики:**
- Нормалізована схема (3NF) для уникнення дублювання
- Багато коротких транзакцій
- Час відповіді: < 100 мс
- Висока конкурентність (тисячі користувачів)

**Приклади систем:**
- Банківські операції
- E-commerce замовлення
- Бронювання квитків
- CRM системи

**Приклади БД:**
- PostgreSQL
- MySQL
- Oracle
- SQL Server

```sql
-- Типовий OLTP запит
INSERT INTO orders (user_id, product_id, quantity, created_at)
VALUES (123, 456, 2, NOW());

UPDATE inventory SET stock = stock - 2 WHERE product_id = 456;
```

---

### OLAP — Online Analytical Processing

**Характеристики:**
- Денормалізована схема (Star/Snowflake) для швидкості читання
- Складні аналітичні запити з JOIN та агрегаціями
- Час відповіді: секунди — хвилини
- Мало користувачів, великі обсяги даних

**Приклади задач:**
- Звіти про продажі за регіонами
- Тренди поведінки користувачів
- Прогнозування та ML
- Business Intelligence дашборди

**Приклади БД:**
- ClickHouse
- Amazon Redshift
- Google BigQuery
- Snowflake
- Apache Druid

```sql
-- Типовий OLAP запит
SELECT 
    region,
    product_category,
    EXTRACT(MONTH FROM order_date) as month,
    SUM(revenue) as total_revenue,
    COUNT(DISTINCT customer_id) as unique_customers
FROM fact_sales
JOIN dim_products USING (product_id)
JOIN dim_regions USING (region_id)
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY region, product_category, month
ORDER BY total_revenue DESC;
```

---

### Star Schema vs Snowflake Schema

**Star Schema:**
```
                    ┌─────────────┐
                    │ dim_product │
                    └──────┬──────┘
                           │
┌─────────────┐    ┌───────┴───────┐    ┌─────────────┐
│  dim_time   │────│  fact_sales   │────│ dim_customer│
└─────────────┘    └───────┬───────┘    └─────────────┘
                           │
                    ┌──────┴──────┐
                    │ dim_region  │
                    └─────────────┘
```

**Snowflake Schema:** dimensions додатково нормалізовані (dim_product → dim_category → dim_brand)

---

## Безпека даних

| Механізм | Опис |
|----------|------|
| **Аутентифікація** | Перевірка особи (логін/пароль, сертифікати) |
| **Авторизація** | Права доступу (GRANT, REVOKE, RBAC) |
| **Шифрування at rest** | Дані зашифровані на диску (TDE) |
| **Шифрування in transit** | SSL/TLS для з'єднань |
| **SQL Injection захист** | Параметризовані запити |
| **Аудит** | Логування всіх операцій |

**SQL Injection захист:**
```python
# ❌ Вразливо
query = f"SELECT * FROM users WHERE email = '{email}'"

# ✅ Безпечно
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

---

## Рівні ізоляції транзакцій

| Рівень | Dirty Read | Non-repeatable Read | Phantom Read |
|--------|:----------:|:-------------------:|:------------:|
| READ UNCOMMITTED | ✓ | ✓ | ✓ |
| READ COMMITTED | ✗ | ✓ | ✓ |
| REPEATABLE READ | ✗ | ✗ | ✓ |
| SERIALIZABLE | ✗ | ✗ | ✗ |

**Проблеми:**
- **Dirty Read:** читання незакомічених змін
- **Non-repeatable Read:** повторне читання дає інший результат
- **Phantom Read:** з'являються нові рядки

---

## Нормалізація

| Форма | Вимога |
|-------|--------|
| **1NF** | Атомарні значення (немає масивів) |
| **2NF** | 1NF + немає часткової залежності |
| **3NF** | 2NF + немає транзитивних залежностей |
| **BCNF** | Кожен детермінант — кандидат у ключі |

---

## Корисні команди

```sql
-- Аналіз плану запиту
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@gmail.com';

-- Створення індексу
CREATE INDEX CONCURRENTLY idx_email ON users(email);

-- Статистика індексів
SELECT * FROM pg_stat_user_indexes WHERE relname = 'users';

-- Розмір таблиці
SELECT pg_size_pretty(pg_total_relation_size('users'));
```