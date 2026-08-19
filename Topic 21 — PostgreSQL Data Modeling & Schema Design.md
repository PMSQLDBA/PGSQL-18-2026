# Topic 21 — PostgreSQL Data Modeling & Schema Design

This topic is about **how to design PostgreSQL tables correctly before writing application queries**. 

A strong schema makes the database easier to maintain, improves data quality, and usually makes performance tuning much simpler.

We’ll cover it from a **developer + technical architecture + interview** point of view.

---

## 1. What is Data Modeling?

Data modeling means deciding:

* What data needs to be stored?
* What tables should exist?
* What columns belong in each table?
* What data type should each column use?
* What uniquely identifies a row?
* How are tables related?
* What rules should the database enforce?

For example, suppose we are building an **e-commerce application**.

We may identify these entities:

```text
Customer
   |
   | 1:N
   v
Order
   |
   | 1:N
   v
Order_Item
   |
   | N:1
   v
Product
```

That could become:

```sql
CREATE TABLE customers (
    customer_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email varchar(255) NOT NULL UNIQUE,
    first_name varchar(100) NOT NULL,
    last_name varchar(100) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE products (
    product_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_name varchar(200) NOT NULL,
    price numeric(12,2) NOT NULL CHECK (price >= 0),
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    order_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL,
    order_status varchar(30) NOT NULL,
    order_date timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_id bigint NOT NULL,
    product_id bigint NOT NULL,
    quantity integer NOT NULL CHECK (quantity > 0),
    unit_price numeric(12,2) NOT NULL CHECK (unit_price >= 0),

    PRIMARY KEY (order_id, product_id),

    FOREIGN KEY (order_id)
        REFERENCES orders(order_id),

    FOREIGN KEY (product_id)
        REFERENCES products(product_id)
);
```

This small example already contains most of the important schema-design concepts.

---

# 2. PostgreSQL Data Types

Choosing the correct data type is one of the most important schema-design decisions.

A good rule is:

> Use the data type that represents the business meaning of the value, not simply the type that can hold it.

For example, don't store everything as `varchar`.

## Common PostgreSQL types

| Requirement                       | Common PostgreSQL Type     |
| --------------------------------- | -------------------------- |
| Small integer                     | `smallint`                 |
| Integer                           | `integer` / `int`          |
| Large integer                     | `bigint`                   |
| Decimal/money-like value          | `numeric(p,s)`             |
| Floating-point value              | `real`, `double precision` |
| Boolean                           | `boolean`                  |
| Variable text                     | `varchar(n)`               |
| General text                      | `text`                     |
| Date only                         | `date`                     |
| Time only                         | `time`                     |
| Timestamp                         | `timestamp`                |
| Timestamp with timezone semantics | `timestamptz`              |
| UUID                              | `uuid`                     |
| JSON document                     | `json` / `jsonb`           |
| Binary data                       | `bytea`                    |
| Array                             | `type[]`                   |
| IP address                        | `inet`                     |
| Network                           | `cidr`                     |

---

# 3. INTEGER vs BIGINT

PostgreSQL commonly uses:

```sql
integer
bigint
```

For example:

```sql
quantity integer
customer_id bigint
```

`integer` is 32-bit while `bigint` is 64-bit.

For IDs in systems expected to grow significantly, `bigint` is commonly chosen:

```sql
customer_id bigint GENERATED ALWAYS AS IDENTITY
```

For small bounded values:

```sql
retry_count smallint
```

Don't automatically use `bigint` for every numeric column. Choose based on the expected domain.

---

# 4. NUMERIC vs FLOAT

For values where exact decimal precision matters, use:

```sql
numeric(12,2)
```

Example:

```sql
price numeric(12,2)
```

For example:

```text
1250.75
```

Avoid floating-point types for exact financial calculations because floating-point values can contain representation/rounding differences.

For measurements or scientific calculations, `double precision` may be appropriate.

```sql
temperature double precision
```

Architecture rule:

```text
Financial/exact decimal → NUMERIC
Approximate scientific value → DOUBLE PRECISION
```

---

# 5. VARCHAR vs TEXT

PostgreSQL supports both:

```sql
varchar(100)
text
```

Example:

```sql
first_name varchar(100)
description text
```

Unlike some databases, PostgreSQL generally doesn't gain a performance advantage merely from using `varchar(n)` instead of `text`.

Use `varchar(n)` when the maximum length represents an actual business rule.

For example:

```sql
country_code varchar(2)
```

Otherwise:

```sql
description text
```

is often perfectly appropriate.

---

# 6. DATE, TIMESTAMP and TIMESTAMPTZ

This is important in production systems.

Use:

```sql
date
```

when you only need a calendar date.

Example:

```sql
birth_date date
```

Use:

```sql
timestamp
```

for a timestamp without timezone interpretation.

Use:

```sql
timestamptz
```

for an actual point in time.

For example:

```sql
created_at timestamptz NOT NULL DEFAULT now()
```

For distributed applications, `timestamptz` is usually the safer choice for events such as:

```text
created_at
updated_at
login_time
payment_time
order_time
```

---

# 7. UUID

PostgreSQL has a native UUID type.

```sql
user_id uuid
```

A UUID looks like:

```text
550e8400-e29b-41d4-a716-446655440000
```

It can be useful when IDs need to be generated independently across distributed systems.

For example:

```sql
CREATE TABLE users (
    user_id uuid PRIMARY KEY,
    email text NOT NULL UNIQUE
);
```

A UUID is much larger than a simple integer ID, so don't choose it merely because it looks more sophisticated.

---

# 8. NULL vs NOT NULL

A very common schema-design mistake is allowing `NULL` everywhere.

Suppose an order must always belong to a customer.

Bad:

```sql
customer_id bigint
```

Better:

```sql
customer_id bigint NOT NULL
```

Similarly:

```sql
email text NOT NULL
```

Think of `NULL` as:

```text
unknown / absent
```

rather than:

```text
empty
```

These are different:

```text
NULL
''
0
false
```

---

# 9. Constraints

Constraints allow PostgreSQL to enforce data quality.

The major constraints are:

```text
PRIMARY KEY
FOREIGN KEY
UNIQUE
NOT NULL
CHECK
EXCLUDE
```

---

# 10. PRIMARY KEY

A primary key uniquely identifies each row.

Example:

```sql
CREATE TABLE customers (
    customer_id bigint PRIMARY KEY,
    customer_name text NOT NULL
);
```

A primary key provides:

```text
Uniqueness
+
NOT NULL
```

and PostgreSQL creates the supporting unique index.

---

# 11. Natural Key vs Surrogate Key

Suppose customers have an email address.

You could technically use:

```sql
PRIMARY KEY (email)
```

This is a **natural key** because the value comes from the business domain.

But many systems instead use:

```sql
customer_id bigint PRIMARY KEY
```

This is a **surrogate key**.

Then:

```sql
email text NOT NULL UNIQUE
```

Architecture:

```text
customer_id → technical identity
email       → business uniqueness
```

This is useful because business attributes can change.

For example:

```text
john@gmail.com
        ↓
john@company.com
```

Changing the email shouldn't require changing every foreign key referencing that customer.

---

# 12. Composite Primary Keys

A primary key can contain multiple columns.

For example:

```sql
CREATE TABLE order_items (
    order_id bigint NOT NULL,
    product_id bigint NOT NULL,

    PRIMARY KEY (order_id, product_id)
);
```

This means the combination must be unique.

These could exist:

```text
order_id | product_id
---------+-----------
100      | 1
100      | 2
101      | 1
```

But this cannot occur twice:

```text
100 | 1
```

Composite keys are particularly useful in relationship/junction tables.

---

# 13. FOREIGN KEY

Foreign keys maintain **referential integrity**.

Example:

```sql
CREATE TABLE orders (
    order_id bigint PRIMARY KEY,
    customer_id bigint NOT NULL,

    FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);
```

This prevents an order from referencing a customer that doesn't exist.

Without the FK, you could accidentally get:

```text
orders.customer_id = 99999

but

customers.customer_id = 99999
doesn't exist
```

That creates orphaned data.

---

# 14. Foreign Key Actions

PostgreSQL lets us define what happens when the referenced row changes.

Common options include:

```sql
ON DELETE CASCADE
ON DELETE RESTRICT
ON DELETE SET NULL
```

Example:

```sql
FOREIGN KEY (order_id)
REFERENCES orders(order_id)
ON DELETE CASCADE
```

Meaning:

```text
Delete Order
     ↓
Automatically delete its order_items
```

This can make sense because an `order_item` may have no meaning without its order.

But be careful with:

```sql
ON DELETE CASCADE
```

A deletion can propagate through multiple tables.

For important business/history data, deletion may instead be prevented or handled through application/business rules.

---

# 15. UNIQUE Constraint

Suppose every customer must have a unique email.

```sql
email text NOT NULL UNIQUE
```

Now PostgreSQL prevents:

```text
customer 1 → abc@example.com
customer 2 → abc@example.com
```

This is much stronger than relying only on application code.

Why?

Two concurrent requests can both do:

```text
Check email
Email doesn't exist
Insert email
```

A database `UNIQUE` constraint handles this race at the database level.

---

# 16. CHECK Constraint

`CHECK` constraints enforce business rules.

Example:

```sql
price numeric(12,2)
    CHECK (price >= 0)
```

Another example:

```sql
quantity integer
    CHECK (quantity > 0)
```

Another:

```sql
discount_percentage numeric(5,2)
    CHECK (
        discount_percentage >= 0
        AND discount_percentage <= 100
    )
```

This prevents invalid data regardless of which application writes to PostgreSQL.

---

# 17. Normalization

Normalization organizes relational data to reduce unnecessary duplication and data anomalies.

The most commonly discussed levels are:

```text
1NF
2NF
3NF
BCNF
```

For most application schema discussions, understanding **1NF–3NF well** is important.

---

# 18. First Normal Form — 1NF

A table should not contain repeating groups or multiple independent values packed into one field.

Bad:

```text
customer_id | customer_name | phone_numbers
------------+---------------+------------------------
1           | John          | 111-1111,222-2222
```

Better:

```text
customers
---------
customer_id
customer_name
```

and:

```text
customer_phones
---------------
customer_id
phone_number
```

Data becomes:

```text
1 | 111-1111
1 | 222-2222
```

Now phone numbers can be queried and constrained individually.

---

# 19. Second Normal Form — 2NF

2NF matters when a table has a composite key.

Non-key attributes should depend on the **whole key**, not only part of it.

Suppose:

```text
order_id
product_id
product_name
quantity
```

Primary key:

```text
(order_id, product_id)
```

But:

```text
product_name
```

depends only on:

```text
product_id
```

not the complete key.

So move product information into:

```text
products
--------
product_id
product_name
```

And keep:

```text
order_items
-----------
order_id
product_id
quantity
```

---

# 20. Third Normal Form — 3NF

Non-key attributes should not depend on other non-key attributes.

Bad design:

```text
employee_id
employee_name
department_id
department_name
```

Here:

```text
employee_id
   ↓
department_id
   ↓
department_name
```

`department_name` belongs to the department.

Better:

```text
employees
---------
employee_id
employee_name
department_id
```

and:

```text
departments
-----------
department_id
department_name
```

---

# 21. Why Normalize?

Normalization reduces:

### Update anomalies

Without normalization:

```text
Employee | Department
John     | Information Technology
Mary     | Information Technology
Bob      | Information Technology
```

If the department name changes, multiple rows must change.

### Insert anomalies

You may be unable to create a department until an employee exists.

### Delete anomalies

Deleting the last employee might accidentally remove the only stored information about the department.

Normalization separates these concepts.

---

# 22. Denormalization

Normalization is not an absolute rule that every schema must maximize.

Sometimes data is deliberately duplicated for:

```text
Reporting
Analytics
Read-heavy systems
Precomputed results
Performance optimization
Historical snapshots
```

Example:

An order item may store:

```sql
unit_price numeric(12,2)
```

even though the product table also contains a current price.

Why?

Because:

```text
products.price
```

may change tomorrow.

But:

```text
order_items.unit_price
```

should represent the price paid when the order was created.

That's intentional duplication with a business reason.

---

# 23. Sequences

PostgreSQL sequences generate numeric values.

Example:

```sql
CREATE SEQUENCE customer_id_seq;
```

Then:

```sql
CREATE TABLE customers (
    customer_id bigint
        DEFAULT nextval('customer_id_seq')
);
```

Calling:

```sql
SELECT nextval('customer_id_seq');
```

could produce:

```text
1
2
3
4
...
```

Important:

> Sequence-generated values are designed for uniqueness/concurrency, not guaranteed gap-free numbering.

You might see:

```text
100
101
103
104
```

because `102` may have been consumed by a transaction that later rolled back.

Don't rely on sequences for legally required gapless numbering.

---

# 24. SERIAL

Older PostgreSQL schemas commonly use:

```sql
bigserial
```

Example:

```sql
CREATE TABLE customers (
    customer_id bigserial PRIMARY KEY
);
```

`serial` is essentially shorthand that historically creates an integer column, sequence, and default expression.

You'll still see it in existing systems.

---

# 25. Identity Columns

Modern PostgreSQL provides SQL-standard identity columns.

```sql
customer_id bigint
    GENERATED ALWAYS AS IDENTITY
```

Example:

```sql
CREATE TABLE customers (
    customer_id bigint
        GENERATED ALWAYS AS IDENTITY
        PRIMARY KEY,

    email text NOT NULL UNIQUE
);
```

Another option is:

```sql
GENERATED BY DEFAULT AS IDENTITY
```

Difference:

```text
ALWAYS
```

PostgreSQL normally generates the value.

```text
BY DEFAULT
```

You can explicitly provide a value when necessary.

For new schemas, identity columns are generally preferable to `serial`.

---

# 26. JSON

PostgreSQL supports:

```sql
json
```

and:

```sql
jsonb
```

Example:

```sql
CREATE TABLE events (
    event_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    payload jsonb NOT NULL
);
```

Insert:

```sql
INSERT INTO events(payload)
VALUES (
    '{
       "event": "order_created",
       "order_id": 1001,
       "source": "mobile"
     }'
);
```

---

# 27. JSON vs JSONB

Both store JSON data, but they behave differently internally.

### JSON

Stores the JSON input largely as text representation.

Useful when preserving the original JSON representation matters.

### JSONB

Stores a decomposed binary representation optimized for processing.

It generally provides better support for:

```text
Searching
Filtering
Containment operations
Indexing
```

For application data that needs querying, `jsonb` is usually the more common choice.

Example:

```sql
SELECT *
FROM events
WHERE payload->>'event' = 'order_created';
```

---

# 28. JSONB Operators

Suppose:

```json
{
  "customer": {
    "name": "John",
    "country": "US"
  },
  "amount": 125.50
}
```

Retrieve JSON:

```sql
SELECT payload->'customer'
FROM events;
```

Retrieve a JSON value as text:

```sql
SELECT payload->>'amount'
FROM events;
```

Nested value:

```sql
SELECT payload->'customer'->>'country'
FROM events;
```

Containment:

```sql
SELECT *
FROM events
WHERE payload @> '{"amount": 125.50}';
```

---

# 29. JSONB Indexing

A major reason to use `jsonb` is indexing.

For example:

```sql
CREATE INDEX idx_events_payload
ON events
USING GIN (payload);
```

This can support appropriate JSONB operations such as containment queries:

```sql
SELECT *
FROM events
WHERE payload @> '{"event":"order_created"}';
```

For a frequently queried specific property, an expression index may be better:

```sql
CREATE INDEX idx_events_event_type
ON events ((payload->>'event'));
```

Then queries like:

```sql
SELECT *
FROM events
WHERE payload->>'event' = 'order_created';
```

can potentially use that index.

---

# 30. Should Everything Be JSONB?

No.

A common PostgreSQL design mistake is:

```sql
CREATE TABLE customers (
    customer_id bigint,
    data jsonb
);
```

with everything inside `data`.

For example:

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "email": "john@example.com",
  "status": "ACTIVE"
}
```

If these are core business attributes, regular columns are usually better:

```sql
CREATE TABLE customers (
    customer_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name text NOT NULL,
    last_name text NOT NULL,
    email text NOT NULL UNIQUE,
    status text NOT NULL
);
```

Why?

Because relational columns give us straightforward:

```text
Constraints
Foreign keys
Data types
Indexes
Querying
Statistics
Schema clarity
```

---

# 31. When JSONB Makes Sense

JSONB is useful when attributes are:

```text
Optional
Dynamic
Semi-structured
Integration-specific
Different between records
```

Example:

```sql
CREATE TABLE products (
    product_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_name text NOT NULL,
    price numeric(12,2) NOT NULL,
    attributes jsonb
);
```

A laptop might have:

```json
{
  "cpu": "Intel",
  "ram": "32GB",
  "screen": "15 inch"
}
```

A shirt might have:

```json
{
  "size": "L",
  "color": "Blue",
  "material": "Cotton"
}
```

The core attributes remain relational:

```text
product_id
product_name
price
```

Flexible category-specific attributes go into:

```text
attributes JSONB
```

That's a good hybrid model.

---

# 32. Relational vs JSONB Decision

Use this simple rule:

| Requirement                    | Prefer            |
| ------------------------------ | ----------------- |
| Primary identifier             | Column            |
| Foreign key                    | Column            |
| Frequently queried attribute   | Usually column    |
| Strong validation required     | Column            |
| Stable business attribute      | Column            |
| Highly dynamic metadata        | JSONB             |
| External API payload           | JSONB             |
| Event payload                  | JSONB             |
| Variable product properties    | JSONB             |
| Relationships between entities | Relational tables |

Architecture principle:

> Use relational modeling for the stable core of the domain and JSONB for genuinely flexible parts.

---

# 33. One-to-One Relationship

Example:

```text
User
 |
 | 1:1
 |
User_Profile
```

Implementation:

```sql
CREATE TABLE user_profiles (
    user_id bigint PRIMARY KEY,
    bio text,

    FOREIGN KEY (user_id)
        REFERENCES users(user_id)
);
```

Because `user_id` is the primary key, each user can have only one profile row.

---

# 34. One-to-Many Relationship

Example:

```text
Customer
   |
   | 1
   |
   | N
 Order
```

Implementation:

```sql
CREATE TABLE orders (
    order_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    customer_id bigint NOT NULL
        REFERENCES customers(customer_id)
);
```

The foreign key belongs on the **many side**.

---

# 35. Many-to-Many Relationship

Example:

```text
Student
   N
   |
   |
   N
Course
```

Relational databases implement this using a junction table:

```text
students
courses
student_courses
```

Example:

```sql
CREATE TABLE student_courses (
    student_id bigint NOT NULL
        REFERENCES students(student_id),

    course_id bigint NOT NULL
        REFERENCES courses(course_id),

    PRIMARY KEY (student_id, course_id)
);
```

Now:

```text
Student A → Course 1
Student A → Course 2
Student B → Course 1
```

is easy to represent.

---

# 36. Schema Design Example

Suppose we're designing an order-management platform.

A reasonable high-level model:

```text
CUSTOMERS
    |
    | 1:N
    v
ORDERS
    |
    | 1:N
    v
ORDER_ITEMS
    |
    | N:1
    v
PRODUCTS
```

Tables:

```sql
CREATE TABLE customers (
    customer_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email text NOT NULL UNIQUE,
    first_name text NOT NULL,
    last_name text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE products (
    product_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku text NOT NULL UNIQUE,
    product_name text NOT NULL,
    current_price numeric(12,2) NOT NULL
        CHECK (current_price >= 0),
    attributes jsonb NOT NULL DEFAULT '{}'::jsonb,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    order_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL
        REFERENCES customers(customer_id),
    status text NOT NULL,
    ordered_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_id bigint NOT NULL
        REFERENCES orders(order_id),

    product_id bigint NOT NULL
        REFERENCES products(product_id),

    quantity integer NOT NULL
        CHECK (quantity > 0),

    unit_price numeric(12,2) NOT NULL
        CHECK (unit_price >= 0),

    PRIMARY KEY (order_id, product_id)
);
```

---

# 37. Important Design Detail: Current vs Historical Data

Notice:

```text
products.current_price
```

and:

```text
order_items.unit_price
```

may contain different values.

Example:

```text
Monday:
Product price = $100

Customer buys product:
Order unit_price = $100

Friday:
Product price changes = $120
```

We should **not** calculate the historical order as:

```text
quantity × products.current_price
```

because that would incorrectly make the old order worth $120.

Instead:

```text
quantity × order_items.unit_price
```

This is an important schema-design principle:

> Store historical facts separately from mutable current-state data.

---

# 38. Foreign Keys and Indexes

A common PostgreSQL interview point:

> Does PostgreSQL automatically create an index on the referencing foreign-key column?

Not simply because you created the FK.

For example:

```sql
FOREIGN KEY (customer_id)
REFERENCES customers(customer_id)
```

The referenced key is indexed because it must be a primary/unique key, but PostgreSQL doesn't automatically create an index on:

```text
orders.customer_id
```

You often want:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

This can help:

```sql
SELECT *
FROM orders
WHERE customer_id = 100;
```

and can also improve operations involving the parent row.

---

# 39. Naming Conventions

Pick one convention and use it consistently.

A simple convention:

```text
customers
orders
order_items
```

Columns:

```text
customer_id
order_id
created_at
updated_at
```

Indexes:

```text
idx_orders_customer_id
idx_orders_ordered_at
```

Constraints:

```text
pk_orders
fk_orders_customer
uq_customers_email
ck_products_price
```

Avoid inconsistent names such as:

```text
CustomerID
customer_id
CustId
customerID
```

in the same system.

Consistency matters more than personal preference.

---

# 40. Avoid Quoted Mixed-Case Identifiers

Avoid:

```sql
CREATE TABLE "CustomerOrder" (
    "OrderId" bigint
);
```

Because you'll need quoted identifiers:

```sql
SELECT "OrderId"
FROM "CustomerOrder";
```

Prefer lowercase `snake_case`:

```sql
CREATE TABLE customer_orders (
    order_id bigint
);
```

Then:

```sql
SELECT order_id
FROM customer_orders;
```

Much easier to maintain.

---

# 41. Don't Store Comma-Separated Relationships

Bad:

```text
customer_id | product_ids
------------+-------------
1           | 10,20,30
```

Now questions like:

```text
Which customers bought product 20?
```

become unnecessarily difficult.

Instead:

```text
customer_products
-----------------
customer_id
product_id
```

Use relational rows for relationships.

---

# 42. Don't Use Meaningful IDs Without a Good Reason

Suppose somebody designs:

```text
ORDER-2026-USA-000001
```

as the primary key.

It contains:

```text
year
country
sequence
```

This can cause problems when business rules change.

Instead:

```text
order_id = 928472
```

can be the technical key.

And:

```text
order_number = ORDER-2026-USA-000001
```

can be a separately constrained business identifier.

For example:

```sql
order_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
order_number text NOT NULL UNIQUE
```

---

# 43. Status Column Design

Suppose:

```sql
status text NOT NULL
```

Without validation, someone can insert:

```text
ACTVE
COMPLETEDDD
xyz
```

One solution is a `CHECK` constraint:

```sql
status text NOT NULL
CHECK (
    status IN (
        'PENDING',
        'PAID',
        'SHIPPED',
        'CANCELLED'
    )
)
```

PostgreSQL enums are another possibility, but they create stronger coupling between the schema and the set of values.

For frequently changing business states, a lookup/reference table can sometimes be more flexible.

The choice depends on the domain.

---

# 44. Schema Design Checklist

When reviewing a new PostgreSQL schema, I would check:

```text
Business entities identified?
        ↓
Correct relationships?
        ↓
Correct PK strategy?
        ↓
Correct FK relationships?
        ↓
Appropriate data types?
        ↓
NOT NULL where required?
        ↓
UNIQUE business rules?
        ↓
CHECK constraints?
        ↓
Normalized appropriately?
        ↓
Historical data preserved?
        ↓
JSONB used only where useful?
        ↓
FK/query indexes considered?
        ↓
Naming consistent?
```

---

# 45. Architecture Example

Imagine a requirement:

> Design PostgreSQL storage for customers who can place orders containing multiple products. Product metadata differs by category.

I would model it as:

```text
customers
---------
customer_id PK
email UNIQUE
name
created_at


products
--------
product_id PK
sku UNIQUE
name
current_price
attributes JSONB


orders
------
order_id PK
customer_id FK
status
ordered_at


order_items
-----------
order_id PK/FK
product_id PK/FK
quantity
unit_price
```

Why?

```text
Customers ↔ Orders
    One-to-Many

Orders ↔ Products
    Many-to-Many

order_items
    resolves the M:N relationship

unit_price
    preserves historical pricing

attributes JSONB
    supports flexible product metadata

email / sku
    business uniqueness

bigint identity
    stable surrogate identifiers
```

That's the type of reasoning expected from a senior developer or architect—not just writing `CREATE TABLE`.

---

# 46. Common Schema Design Mistakes

Watch for these in real projects and interviews:

1. **Using `varchar` for everything**
2. **Allowing `NULL` on mandatory attributes**
3. **Missing primary keys**
4. **Missing foreign keys because "the application handles it"**
5. **Missing unique constraints**
6. **Using `float` for exact financial values**
7. **Storing comma-separated IDs**
8. **Putting the entire domain model into JSONB**
9. **Using JSONB for relationships**
10. **Over-normalizing without considering query patterns**
11. **Denormalizing without a measurable reason**
12. **Using current values when historical values must be preserved**
13. **Assuming foreign keys automatically create indexes on referencing columns**
14. **Using business attributes as PKs when they can change**
15. **Using sequences while expecting gap-free numbers**
16. **Using `timestamp` when the value represents a global point in time**
17. **Using cascading deletes without understanding the impact**

---

# 47. Interview Questions You Should Be Ready For

**Q: `SERIAL` vs `IDENTITY`?**

`SERIAL` is older PostgreSQL shorthand based on sequences. Identity columns are SQL-standard and are generally preferred for new schema designs.

```sql
id bigint GENERATED ALWAYS AS IDENTITY
```

---

**Q: `JSON` vs `JSONB`?**

`JSON` preserves the input representation, while `JSONB` stores a processed binary representation and supports powerful indexing/query operations. For queryable application JSON, `JSONB` is usually preferred.

---

**Q: Primary key vs unique constraint?**

A primary key is the main row identifier and is implicitly `NOT NULL` and unique. A table has one primary key, though it can contain multiple columns. Multiple unique constraints can exist.

---

**Q: Why use foreign keys?**

To maintain referential integrity and prevent invalid relationships/orphaned references.

---

**Q: Why normalize?**

To reduce unnecessary duplication and avoid update, insert, and delete anomalies.

---

**Q: Should every schema be fully normalized?**

No. Normalize based on correctness first, then selectively denormalize when there is a clear business or performance reason.

---

**Q: Why use a surrogate key?**

It gives the row a stable technical identity independent of business attributes that may change.

---

**Q: Why isn't `customer_id` automatically indexed because it's an FK?**

PostgreSQL requires the referenced target to be backed by an appropriate unique/primary key, but doesn't automatically create an index on the referencing FK column. Add one when query patterns and parent-table operations justify it.

---

# 48. Senior-Level Interview Scenario

### Question

You have a customer table containing 200 million customers. Each customer can have multiple addresses. How would you model it?

A solid starting design:

```sql
CREATE TABLE customers (
    customer_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email text NOT NULL UNIQUE,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE customer_addresses (
    address_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    customer_id bigint NOT NULL
        REFERENCES customers(customer_id),

    address_type text NOT NULL,

    address_line1 text NOT NULL,
    city text NOT NULL,
    postal_code text,
    country_code varchar(2) NOT NULL
);

CREATE INDEX idx_customer_addresses_customer
ON customer_addresses(customer_id);
```

Reasoning:

```text
Don't put:

address1
address2
address3

inside customers.
```

Because the number of addresses is a **one-to-many relationship**.

Also don't do:

```json
addresses JSONB
```

by default if addresses are core business entities that need independent querying, constraints, indexing, updates, or relationships.

Instead:

```text
customers
    |
    | 1:N
    v
customer_addresses
```

That's clean relational modeling.

---

# 49. The Mental Model to Remember

Think about PostgreSQL schema design in this order:

```text
1. Understand business entities
          ↓
2. Identify relationships
          ↓
3. Normalize the core model
          ↓
4. Choose stable primary keys
          ↓
5. Define foreign keys
          ↓
6. Choose correct data types
          ↓
7. Add NOT NULL / UNIQUE / CHECK
          ↓
8. Decide what historical data must survive changes
          ↓
9. Use JSONB only for flexible data
          ↓
10. Add indexes based on access patterns
          ↓
11. Denormalize only when justified
```

The key architecture idea for **Topic 21** is:

> **Design PostgreSQL around business entities, relationships, integrity, and access patterns. Keep stable and important business data relational, enforce rules with constraints, use PK/FK relationships explicitly, use identity/sequences for generated IDs, and use JSONB as a controlled extension for flexible data—not as a replacement for relational modeling.**
