# Django ORM Revision Notes

## 1. Model Basics

### What a Model Is

A **Django model** is a Python class representing a **database table**.

Mapping:

| Django   | Database |
| -------- | -------- |
| Model    | Table    |
| Field    | Column   |
| Instance | Row      |

Example:

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=100)
    price = models.IntegerField()
```

Table generated:

```
product
-------
id
name
price
```

---

### Default Primary Key

If not defined:

```
id = BigAutoField(primary_key=True)
```

Auto-increment integer.

---

### Creating Objects

```python
product = Product(name="Phone", price=500)
product.save()
```

Save triggers:

```
INSERT or UPDATE
```

Decision rule:

```
pk is None → INSERT
pk exists → UPDATE
```

---

### Deleting Objects

```python
product.delete()
```

SQL:

```
DELETE FROM product WHERE id = ...
```

Bulk delete:

```python
Product.objects.filter(price=0).delete()
```

---

### Model Manager

Every model has:

```python
Product.objects
```

This is a **Manager** that returns **QuerySets**.

---

### Model Metadata

All model information is stored in:

```
Model._meta
```

Examples:

```python
Product._meta.fields
Product._meta.db_table
```

Used internally by:

```
ORM
Admin
Migrations
Forms
```

---

# 2. Model Fields

Fields define **data type + validation + database column type**.

All fields inherit from:

```
django.db.models.Field
```

---

### Common Field Types

| Field         | SQL Type         | Use               |
| ------------- | ---------------- | ----------------- |
| CharField     | VARCHAR          | short text        |
| TextField     | TEXT             | long text         |
| IntegerField  | INTEGER          | numbers           |
| DecimalField  | NUMERIC          | money             |
| FloatField    | DOUBLE           | scientific values |
| BooleanField  | BOOLEAN          | true/false        |
| DateField     | DATE             | dates             |
| DateTimeField | TIMESTAMP        | datetime          |
| EmailField    | VARCHAR          | email validation  |
| JSONField     | JSONB (Postgres) | structured data   |

Example:

```python
price = models.DecimalField(
    max_digits=10,
    decimal_places=2
)
```

---

### DecimalField Parameters

```
max_digits → total digits
decimal_places → digits after decimal
```

Example:

```
99999999.99
```

---

### Python ↔ Database Conversion

Fields implement:

```
to_python()
from_db_value()
get_prep_value()
```

These convert values between Python and SQL.

---

# 3. Field Options

Field options modify **database schema + validation behavior**.

Example:

```python
price = models.IntegerField(null=True, default=0)
```

---

### `null`

Controls **database nullability**.

```
null=True → column allows NULL
```

---

### `blank`

Controls **form validation**.

```
blank=True → form allows empty input
```

---

### `default`

Default value when none provided.

```python
stock = models.IntegerField(default=0)
```

Callable allowed:

```python
default=uuid4
```

---

### `unique`

Creates **database UNIQUE constraint**.

```python
email = models.EmailField(unique=True)
```

---

### `db_index`

Creates database index.

```python
email = models.EmailField(db_index=True)
```

Indexes improve:

```
SELECT queries
```

But slow:

```
INSERT / UPDATE
```

---

### `choices`

Restricts values.

```python
STATUS = [
 ("pending","Pending"),
 ("done","Done")
]

status = models.CharField(
    max_length=10,
    choices=STATUS
)
```

---

### Timestamp Options

```
auto_now → update every save
auto_now_add → set once at creation
```

---

# 4. Model Relationships

Relationships represent **table joins**.

Types:

| Field           | Relationship |
| --------------- | ------------ |
| ForeignKey      | One-to-Many  |
| OneToOneField   | One-to-One   |
| ManyToManyField | Many-to-Many |

---

# ForeignKey

Example:

```python
class Book(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
```

Database column:

```
author_id
```

---

### Reverse Relation

Default:

```
author.book_set
```

Better:

```python
related_name="books"
```

Then:

```
author.books.all()
```

---

### on_delete Options

| Option      | Behavior         |
| ----------- | ---------------- |
| CASCADE     | delete children  |
| PROTECT     | prevent deletion |
| SET_NULL    | set NULL         |
| SET_DEFAULT | set default      |
| DO_NOTHING  | ignore           |

---

# OneToOneField

Example:

```python
class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
```

Creates:

```
UNIQUE(user_id)
```

Reverse relation:

```
user.profile
```

---

# ManyToManyField

Example:

```python
class Student(models.Model):
    courses = models.ManyToManyField(Course)
```

Creates **join table**:

```
student_courses
--------------
student_id
course_id
```

---

# 5. Model Meta Options

`Meta` defines **table-level configuration**.

Example:

```python
class Meta:
    ordering = ["price"]
```

---

### ordering

Default query order.

```
ordering = ["price"]
ordering = ["-price"]
```

---

### db_table

Custom table name.

```python
db_table = "products"
```

---

### indexes

Custom database indexes.

```python
indexes = [
    models.Index(fields=["name"])
]
```

---

### constraints

Database rules.

Example:

```python
CheckConstraint(
    check=Q(price__gte=0)
)
```

---

### unique_together

Composite uniqueness.

```python
unique_together = ["student","course"]
```

Modern alternative:

```
UniqueConstraint
```

---

### abstract

Base models without tables.

```python
class Meta:
    abstract = True
```

---

### managed

```
managed=False
```

Django will not manage migrations.

---

# 6. QuerySets

QuerySet = **lazy database query object**.

Example:

```python
Product.objects.all()
```

At creation:

```
NO SQL executed
```

---

### QuerySet evaluation triggers

Query runs when:

```
iteration
list()
len()
bool()
index access
aggregation
```

---

### Filtering

```python
Product.objects.filter(price__gt=100)
```

Lookup syntax:

```
field__lookup=value
```

Common lookups:

```
gt, lt, gte, lte
contains
icontains
in
range
```

---

### Exclude

```python
Product.objects.exclude(price__lt=100)
```

---

### QuerySet chaining

```python
Product.objects.filter(price__gt=100).filter(stock__gt=0)
```

QuerySets are **immutable**.

Each call returns a **new QuerySet**.

---

### QuerySet caching

After evaluation:

```
results cached in QuerySet._result_cache
```

---

# 7. Query Optimization Tools

## select_related

Used for:

```
ForeignKey
OneToOne
```

Example:

```python
Book.objects.select_related("author")
```

Creates **SQL JOIN**.

Fixes **N+1 problem**.

---

## prefetch_related

Used for:

```
ManyToMany
Reverse ForeignKey
```

Example:

```python
Author.objects.prefetch_related("books")
```

Runs **2 queries**, then joins in Python.

---

# values()

Returns dictionaries instead of model objects.

```python
Product.objects.values("name","price")
```

Result:

```
{"name": "...", "price": ...}
```

---

# values_list()

Returns tuples.

```python
Product.objects.values_list("name","price")
```

Result:

```
("Phone", 500)
```

Flat mode:

```python
values_list("name", flat=True)
```

---

# 8. Advanced ORM

These features allow **complex SQL logic**.

---

# F() Expressions

Reference fields inside queries.

Example:

```python
Product.objects.update(
    stock=F("stock") + 1
)
```

SQL:

```
stock = stock + 1
```

Advantages:

```
atomic
single query
no race condition
```

---

# Q() Objects

Used for complex logical queries.

Example:

```python
Product.objects.filter(
    Q(price__lt=100) | Q(stock__lt=5)
)
```

Operators:

```
| → OR
& → AND
~ → NOT
```

---

# annotate()

Adds computed fields.

Example:

```python
Author.objects.annotate(
    book_count=Count("book")
)
```

Adds:

```
book_count column
```

---

# aggregate()

Returns summary values.

Example:

```python
Product.objects.aggregate(
    avg_price=Avg("price")
)
```

Result:

```
{"avg_price": 350}
```

Common functions:

```
Count
Sum
Avg
Max
Min
```

---

# Subquery

Nested query inside another query.

Example:

```python
Subquery(...)
OuterRef(...)
```

Used when JOIN is insufficient.

---

# Exists

Efficient existence checks.

```python
Exists(
    Review.objects.filter(book=OuterRef("pk"))
)
```

SQL concept:

```
EXISTS(...)
```

---

# Database Functions

Located in:

```
django.db.models.functions
```

Examples:

```
Lower
Upper
Length
Concat
Coalesce
ExtractYear
```

Example:

```python
annotate(name_lower=Lower("name"))
```

---

# Conditional Expressions

SQL CASE equivalent.

Example:

```python
Case(
    When(price__gt=1000, then=Value("expensive")),
    default=Value("cheap")
)
```

---

# Core ORM Mental Model

Django ORM pipeline:

```
QuerySet
   ↓
Query object
   ↓
SQL compiler
   ↓
SQL query
   ↓
Database execution
   ↓
Rows returned
   ↓
Converted to model instances
```

---

# Key Performance Rules

1️⃣ Avoid **N+1 queries**
Use:

```
select_related
prefetch_related
```

2️⃣ Use **database operations instead of Python loops**

```
F()
annotations
aggregations
```

3️⃣ Use **indexes for frequently filtered fields**

4️⃣ Inspect SQL when debugging

```
print(qs.query)
```
