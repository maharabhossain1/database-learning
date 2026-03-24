# Complete Django ORM Mastery Guide

**From SQL to Django ORM - Deep Understanding with Performance Optimization**

> A comprehensive guide for backend engineers to master Django ORM with SQL comparisons, solving N+1 problems, transactions (ACID), optimization techniques, and real-world patterns.

---

## Table of Contents

1. [Django ORM Basics](#1-django-orm-basics)
2. [CRUD Operations - SQL vs ORM](#2-crud-operations)
3. [Querying & Filtering](#3-querying--filtering)
4. [Understanding Relationships](#4-understanding-relationships)
5. [The N+1 Problem & Solutions](#5-the-n1-problem--solutions)
6. [Aggregation & Annotation](#6-aggregation--annotation)
7. [Advanced Queries](#7-advanced-queries)
8. [Transactions & ACID](#8-transactions--acid)
9. [Performance Optimization](#9-performance-optimization)
10. [Database Indexing in Django](#10-database-indexing-in-django)
11. [Real-World Patterns (GTAF Context)](#11-real-world-patterns)
12. [Common Pitfalls & Solutions](#12-common-pitfalls--solutions)
13. [Query Debugging & Monitoring](#13-query-debugging--monitoring)

---

## 1. Django ORM Basics

### 1.1 What is Django ORM?

Django ORM (Object-Relational Mapping) converts Python code into SQL queries. Instead of writing raw SQL, you work with Python objects.

**The Magic Translation:**

```python
# Django ORM
User.objects.filter(age__gt=25)

# Becomes SQL
# SELECT * FROM users WHERE age > 25;
```

### 1.2 Models = Database Tables

```python
# models.py
from django.db import models

class User(models.Model):
    username = models.CharField(max_length=50, unique=True)
    email = models.EmailField(unique=True)
    age = models.IntegerField(null=True, blank=True)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'users'
        indexes = [
            models.Index(fields=['email']),
            models.Index(fields=['username', 'is_active']),
        ]

    def __str__(self):
        return self.username
```

**SQL Equivalent:**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(254) UNIQUE NOT NULL,
    age INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username_active ON users(username, is_active);
```

### 1.3 Field Types Reference

```python
# Common Django Field Types → PostgreSQL Types

models.CharField(max_length=100)        # VARCHAR(100)
models.TextField()                       # TEXT
models.IntegerField()                    # INTEGER
models.BigIntegerField()                 # BIGINT
models.DecimalField(max_digits=10, decimal_places=2)  # DECIMAL(10,2)
models.FloatField()                      # DOUBLE PRECISION
models.BooleanField()                    # BOOLEAN
models.DateField()                       # DATE
models.DateTimeField()                   # TIMESTAMP
models.JSONField()                       # JSONB (PostgreSQL)
models.EmailField()                      # VARCHAR(254)
models.URLField()                        # VARCHAR(200)
models.UUIDField()                       # UUID
```

---

## 2. CRUD Operations

### 2.1 CREATE - Adding Records

#### Single Object Creation

**Django ORM:**

```python
# Method 1: Create and save
user = User(username='maharab', email='maharab@gtaf.org', age=25)
user.save()

# Method 2: Create in one line
user = User.objects.create(
    username='maharab',
    email='maharab@gtaf.org',
    age=25
)

# Method 3: Get or create (avoid duplicates)
user, created = User.objects.get_or_create(
    email='maharab@gtaf.org',
    defaults={'username': 'maharab', 'age': 25}
)
```

**SQL Equivalent:**

```sql
-- Method 1 & 2
INSERT INTO users (username, email, age, is_active, created_at, updated_at)
VALUES ('maharab', 'maharab@gtaf.org', 25, TRUE, NOW(), NOW())
RETURNING id;

-- Method 3 (get_or_create)
INSERT INTO users (username, email, age, is_active, created_at, updated_at)
VALUES ('maharab', 'maharab@gtaf.org', 25, TRUE, NOW(), NOW())
ON CONFLICT (email) DO NOTHING
RETURNING id;
```

#### Bulk Creation (Performance!)

**Django ORM:**

```python
# MUCH faster for multiple records
users = [
    User(username='user1', email='user1@example.com', age=25),
    User(username='user2', email='user2@example.com', age=30),
    User(username='user3', email='user3@example.com', age=35),
]
User.objects.bulk_create(users, batch_size=500)

# With ignore_conflicts (PostgreSQL 9.5+)
User.objects.bulk_create(users, ignore_conflicts=True)

# With update on conflict (PostgreSQL)
User.objects.bulk_create(
    users,
    update_conflicts=True,
    unique_fields=['email'],
    update_fields=['username', 'age']
)
```

**SQL Equivalent:**

```sql
-- bulk_create
INSERT INTO users (username, email, age, is_active, created_at, updated_at)
VALUES
    ('user1', 'user1@example.com', 25, TRUE, NOW(), NOW()),
    ('user2', 'user2@example.com', 30, TRUE, NOW(), NOW()),
    ('user3', 'user3@example.com', 35, TRUE, NOW(), NOW());
```

### 2.2 READ - Retrieving Records

#### Basic Queries

**Django ORM:**

```python
# Get all records
users = User.objects.all()

# Get filtered records
active_users = User.objects.filter(is_active=True)

# Get single record
user = User.objects.get(id=1)  # Raises DoesNotExist if not found

# Get or None (safer)
user = User.objects.filter(username='maharab').first()

# Count
count = User.objects.filter(is_active=True).count()

# Exists (efficient check)
exists = User.objects.filter(email='test@example.com').exists()
```

**SQL Equivalent:**

```sql
-- all()
SELECT * FROM users;

-- filter()
SELECT * FROM users WHERE is_active = TRUE;

-- get()
SELECT * FROM users WHERE id = 1 LIMIT 1;

-- first()
SELECT * FROM users WHERE username = 'maharab' LIMIT 1;

-- count()
SELECT COUNT(*) FROM users WHERE is_active = TRUE;

-- exists()
SELECT EXISTS(SELECT 1 FROM users WHERE email = 'test@example.com' LIMIT 1);
```

#### Selecting Specific Fields

**Django ORM:**

```python
# values() - returns dictionaries
users = User.objects.values('id', 'username', 'email')
# [{'id': 1, 'username': 'maharab', 'email': '...'}]

# values_list() - returns tuples
users = User.objects.values_list('id', 'username')
# [(1, 'maharab'), (2, 'john')]

# Flat list (single field)
usernames = User.objects.values_list('username', flat=True)
# ['maharab', 'john', 'jane']

# only() - loads only specified fields (returns models)
users = User.objects.only('username', 'email')

# defer() - excludes specified fields
users = User.objects.defer('created_at', 'updated_at')
```

**SQL Equivalent:**

```sql
-- values()
SELECT id, username, email FROM users;

-- values_list()
SELECT id, username FROM users;

-- values_list(flat=True)
SELECT username FROM users;

-- only() / defer()
SELECT id, username, email FROM users;
```

### 2.3 UPDATE - Modifying Records

**Django ORM:**

```python
# Update single object
user = User.objects.get(id=1)
user.age = 26
user.save()

# Update specific fields only
user.save(update_fields=['age'])

# Bulk update (one query!)
User.objects.filter(is_active=False).update(is_active=True)

# Update with F expressions (database-level)
from django.db.models import F
User.objects.filter(is_active=True).update(age=F('age') + 1)

# Update or create
user, created = User.objects.update_or_create(
    username='maharab',
    defaults={'age': 26, 'email': 'maharab@gtaf.org'}
)
```

**SQL Equivalent:**

```sql
-- save()
UPDATE users
SET age = 26, updated_at = NOW()
WHERE id = 1;

-- filter().update()
UPDATE users
SET is_active = TRUE
WHERE is_active = FALSE;

-- F expressions
UPDATE users
SET age = age + 1
WHERE is_active = TRUE;
```

### 2.4 DELETE - Removing Records

**Django ORM:**

```python
# Delete single object
user = User.objects.get(id=1)
user.delete()

# Bulk delete
User.objects.filter(is_active=False).delete()

# Soft delete (recommended)
from django.utils import timezone
User.objects.filter(id=1).update(deleted_at=timezone.now())
```

**SQL Equivalent:**

```sql
-- delete()
DELETE FROM users WHERE id = 1;

-- bulk delete
DELETE FROM users WHERE is_active = FALSE;
```

---

## 3. Querying & Filtering

### 3.1 Field Lookups (WHERE Conditions)

**Django ORM:**

```python
# Exact match
User.objects.filter(username='maharab')
User.objects.filter(username__exact='maharab')

# Case-insensitive
User.objects.filter(username__iexact='MAHARAB')

# Contains
User.objects.filter(email__contains='gmail')
User.objects.filter(email__icontains='GMAIL')

# Starts/Ends with
User.objects.filter(username__startswith='mah')
User.objects.filter(email__endswith='@gmail.com')

# Greater than / Less than
User.objects.filter(age__gt=25)   # >
User.objects.filter(age__gte=25)  # >=
User.objects.filter(age__lt=30)   # <
User.objects.filter(age__lte=30)  # <=

# IN lookup
User.objects.filter(id__in=[1, 2, 3, 4])

# Range
User.objects.filter(age__range=(25, 30))

# IS NULL
User.objects.filter(age__isnull=True)
User.objects.filter(age__isnull=False)

# Date lookups
from datetime import date
User.objects.filter(created_at__date=date.today())
User.objects.filter(created_at__year=2024)
User.objects.filter(created_at__month=12)
```

**SQL Equivalent:**

```sql
-- exact
SELECT * FROM users WHERE username = 'maharab';

-- iexact
SELECT * FROM users WHERE LOWER(username) = LOWER('MAHARAB');

-- contains
SELECT * FROM users WHERE email LIKE '%gmail%';

-- icontains
SELECT * FROM users WHERE email ILIKE '%GMAIL%';

-- startswith
SELECT * FROM users WHERE username LIKE 'mah%';

-- gt, gte, lt, lte
SELECT * FROM users WHERE age > 25;
SELECT * FROM users WHERE age >= 25;

-- in
SELECT * FROM users WHERE id IN (1, 2, 3, 4);

-- range
SELECT * FROM users WHERE age BETWEEN 25 AND 30;

-- isnull
SELECT * FROM users WHERE age IS NULL;
SELECT * FROM users WHERE age IS NOT NULL;
```

### 3.2 Complex Filters (AND, OR, NOT)

**Django ORM:**

```python
from django.db.models import Q

# AND (default)
User.objects.filter(is_active=True, age__gt=25)

# OR using Q objects
User.objects.filter(Q(age__lt=18) | Q(age__gt=65))

# Complex OR and AND
User.objects.filter(
    Q(is_active=True) & (Q(age__lt=18) | Q(age__gt=65))
)

# NOT using ~
User.objects.filter(~Q(username__startswith='test'))

# Exclude (NOT)
User.objects.exclude(username__startswith='test')
```

**SQL Equivalent:**

```sql
-- AND
SELECT * FROM users WHERE is_active = TRUE AND age > 25;

-- OR
SELECT * FROM users WHERE age < 18 OR age > 65;

-- Complex
SELECT * FROM users
WHERE is_active = TRUE AND (age < 18 OR age > 65);

-- NOT
SELECT * FROM users WHERE username NOT LIKE 'test%';
```

### 3.3 Ordering (ORDER BY)

**Django ORM:**

```python
# Order by single field
User.objects.order_by('age')           # ASC
User.objects.order_by('-age')          # DESC

# Multiple fields
User.objects.order_by('is_active', '-created_at')

# Random order
User.objects.order_by('?')

# Using F expressions
from django.db.models import F
User.objects.order_by(F('age').desc(nulls_last=True))
```

**SQL Equivalent:**

```sql
-- order_by('age')
SELECT * FROM users ORDER BY age ASC;

-- order_by('-age')
SELECT * FROM users ORDER BY age DESC;

-- Multiple fields
SELECT * FROM users ORDER BY is_active, created_at DESC;

-- Random
SELECT * FROM users ORDER BY RANDOM();

-- nulls_last
SELECT * FROM users ORDER BY age DESC NULLS LAST;
```

### 3.4 Limiting Results (LIMIT/OFFSET)

**Django ORM:**

```python
# First 10 records
users = User.objects.all()[:10]

# Skip 20, get next 10 (pagination)
users = User.objects.all()[20:30]

# First record
user = User.objects.first()

# Last record
user = User.objects.last()

# Specific record by index
user = User.objects.all()[5]  # 6th record (0-indexed)
```

**SQL Equivalent:**

```sql
-- [:10]
SELECT * FROM users LIMIT 10;

-- [20:30]
SELECT * FROM users LIMIT 10 OFFSET 20;

-- first()
SELECT * FROM users ORDER BY id LIMIT 1;

-- last()
SELECT * FROM users ORDER BY id DESC LIMIT 1;
```

---

## 4. Understanding Relationships

### 4.1 One-to-Many (ForeignKey)

**Django Models:**

```python
class Category(models.Model):
    name = models.CharField(max_length=100)

    def __str__(self):
        return self.name

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE,  # Delete products when category deleted
        related_name='products'     # Access products from category
    )

    def __str__(self):
        return self.name
```

**SQL Equivalent:**

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    category_id INTEGER NOT NULL REFERENCES categories(id) ON DELETE CASCADE
);

CREATE INDEX idx_products_category ON products(category_id);
```

**Querying Relationships:**

**Django ORM:**

```python
# Forward relation (Product → Category)
product = Product.objects.get(id=1)
category_name = product.category.name  # Accesses related category

# Reverse relation (Category → Products)
category = Category.objects.get(id=1)
products = category.products.all()  # Uses related_name

# Filter by related field
products = Product.objects.filter(category__name='Electronics')
products = Product.objects.filter(category__id=1)

# Reverse filtering
categories = Category.objects.filter(products__price__gt=100)
```

**SQL Equivalent:**

```sql
-- Forward relation
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.id = 1;

-- Reverse relation
SELECT p.*
FROM products p
WHERE p.category_id = 1;

-- Filter by related field
SELECT p.*
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE c.name = 'Electronics';

-- Reverse filtering
SELECT DISTINCT c.*
FROM categories c
JOIN products p ON c.id = p.category_id
WHERE p.price > 100;
```

### 4.2 Many-to-Many (ManyToManyField)

**Django Models:**

```python
class Student(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)

class Course(models.Model):
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    students = models.ManyToManyField(
        Student,
        related_name='courses',
        through='Enrollment'  # Custom through table
    )

class Enrollment(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    course = models.ForeignKey(Course, on_delete=models.CASCADE)
    enrolled_at = models.DateTimeField(auto_now_add=True)
    grade = models.CharField(max_length=2, null=True)

    class Meta:
        unique_together = ('student', 'course')
```

**SQL Equivalent:**

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(254) UNIQUE NOT NULL
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) UNIQUE NOT NULL
);

CREATE TABLE enrollments (
    id SERIAL PRIMARY KEY,
    student_id INTEGER NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id INTEGER NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMP NOT NULL,
    grade VARCHAR(2),
    UNIQUE(student_id, course_id)
);
```

**Querying Many-to-Many:**

**Django ORM:**

```python
# Add relationships
student = Student.objects.get(id=1)
course = Course.objects.get(id=1)

# Simple add
course.students.add(student)

# Add with through model
Enrollment.objects.create(student=student, course=course, grade='A')

# Get all courses for a student
courses = student.courses.all()

# Get all students in a course
students = course.students.all()

# Filter by related many-to-many
students = Student.objects.filter(courses__code='CS101')

# Count related objects
course_count = student.courses.count()
```

**SQL Equivalent:**

```sql
-- Add relationship
INSERT INTO enrollments (student_id, course_id, enrolled_at)
VALUES (1, 1, NOW());

-- Get all courses for student
SELECT c.*
FROM courses c
JOIN enrollments e ON c.id = e.course_id
WHERE e.student_id = 1;

-- Get all students in course
SELECT s.*
FROM students s
JOIN enrollments e ON s.id = e.student_id
WHERE e.course_id = 1;

-- Filter by related
SELECT DISTINCT s.*
FROM students s
JOIN enrollments e ON s.id = e.student_id
JOIN courses c ON e.course_id = c.id
WHERE c.code = 'CS101';
```

### 4.3 One-to-One (OneToOneField)

**Django Models:**

```python
class User(models.Model):
    username = models.CharField(max_length=50, unique=True)
    email = models.EmailField(unique=True)

class Profile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True)
    date_of_birth = models.DateField(null=True)
```

**SQL Equivalent:**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(254) UNIQUE NOT NULL
);

CREATE TABLE profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar VARCHAR(100),
    date_of_birth DATE
);
```

**Querying One-to-One:**

**Django ORM:**

```python
# Access profile from user
user = User.objects.get(id=1)
bio = user.profile.bio

# Access user from profile
profile = Profile.objects.get(id=1)
username = profile.user.username

# Create with one-to-one
user = User.objects.create(username='maharab', email='maharab@gtaf.org')
Profile.objects.create(user=user, bio='Backend Engineer')
```

---

## 5. The N+1 Problem & Solutions

### 5.1 Understanding the N+1 Problem

**The Problem:**

```python
# This looks innocent but causes N+1 queries!
categories = Category.objects.all()  # 1 query

for category in categories:
    print(category.name)
    print(category.products.count())  # N additional queries!
```

**SQL Executed:**

```sql
-- Query 1: Get all categories
SELECT * FROM categories;

-- Query 2: Count products for category 1
SELECT COUNT(*) FROM products WHERE category_id = 1;

-- Query 3: Count products for category 2
SELECT COUNT(*) FROM products WHERE category_id = 2;

-- This is VERY slow!
```

### 5.2 Solution 1: select_related() - For ForeignKey & OneToOne

**Use when:** Following forward ForeignKey or OneToOne relationships

**Django ORM:**

```python
# ❌ BAD: N+1 Problem
products = Product.objects.all()
for product in products:
    print(product.category.name)  # New query per product!

# ✅ GOOD: Use select_related()
products = Product.objects.select_related('category').all()
for product in products:
    print(product.category.name)  # No extra queries!

# Multiple relations
products = Product.objects.select_related('category', 'manufacturer')

# Chained relations
orders = Order.objects.select_related('user__profile')
```

**SQL Equivalent:**

```sql
-- select_related uses JOIN

-- select_related('category')
SELECT p.*, c.*
FROM products p
INNER JOIN categories c ON p.category_id = c.id;

-- select_related('category', 'manufacturer')
SELECT p.*, c.*, m.*
FROM products p
INNER JOIN categories c ON p.category_id = c.id
INNER JOIN manufacturers m ON p.manufacturer_id = m.id;
```

### 5.3 Solution 2: prefetch_related() - For Reverse ForeignKey & ManyToMany

**Use when:** Following reverse ForeignKey, ManyToMany, or GenericForeignKey

**Django ORM:**

```python
# ❌ BAD: N+1 Problem
categories = Category.objects.all()
for category in categories:
    for product in category.products.all():  # New query per category!
        print(product.name)

# ✅ GOOD: Use prefetch_related()
categories = Category.objects.prefetch_related('products').all()
for category in categories:
    for product in category.products.all():  # No additional queries!
        print(product.name)

# Multiple relations
students = Student.objects.prefetch_related('courses', 'enrollments')

# Chained prefetch
categories = Category.objects.prefetch_related('products__manufacturer')
```

**SQL Equivalent:**

```sql
-- prefetch_related uses separate queries

-- Query 1: Get categories
SELECT * FROM categories;

-- Query 2: Get all related products in one query
SELECT * FROM products WHERE category_id IN (1, 2, 3, 4, 5);

-- Django combines them in Python (more efficient than N queries)
```

### 5.4 Solution 3: Prefetch() for Custom Queries

**Django ORM:**

```python
from django.db.models import Prefetch

# Prefetch with filtering
categories = Category.objects.prefetch_related(
    Prefetch(
        'products',
        queryset=Product.objects.filter(price__gt=100).order_by('-price')
    )
)

# Prefetch with select_related (combine both!)
categories = Category.objects.prefetch_related(
    Prefetch(
        'products',
        queryset=Product.objects.select_related('manufacturer')
    )
)

# Multiple custom prefetches
Student.objects.prefetch_related(
    Prefetch('courses', queryset=Course.objects.filter(is_active=True)),
    Prefetch('enrollments', queryset=Enrollment.objects.filter(grade='A'))
)
```

### 5.5 Performance Comparison

```python
# Example: Get 100 products with their categories

# ❌ BAD: N+1 Problem (101 queries)
products = Product.objects.all()[:100]
for product in products:
    print(product.category.name)
# 1 query for products + 100 queries for categories = 101 queries

# ✅ GOOD: select_related (1 query)
products = Product.objects.select_related('category').all()[:100]
for product in products:
    print(product.category.name)
# 1 query total

# Example: Get 10 categories with their products

# ❌ BAD: N+1 Problem (11 queries)
categories = Category.objects.all()[:10]
for category in categories:
    for product in category.products.all():
        print(product.name)
# 1 query for categories + 10 queries for products = 11 queries

# ✅ GOOD: prefetch_related (2 queries)
categories = Category.objects.prefetch_related('products').all()[:10]
for category in categories:
    for product in category.products.all():
        print(product.name)
# 2 queries total (1 for categories, 1 for all products)
```

---

## 6. Aggregation & Annotation

### 6.1 Aggregate Functions (Single Result)

**Django ORM:**

```python
from django.db.models import Count, Sum, Avg, Min, Max

# COUNT
total_users = User.objects.count()
active_count = User.objects.filter(is_active=True).count()

# Using aggregate (returns dict)
result = User.objects.aggregate(
    total=Count('id'),
    avg_age=Avg('age'),
    min_age=Min('age'),
    max_age=Max('age')
)
# {'total': 100, 'avg_age': 28.5, 'min_age': 18, 'max_age': 65}

# SUM
total_price = Product.objects.aggregate(total=Sum('price'))

# Multiple aggregates
stats = Product.objects.aggregate(
    total_products=Count('id'),
    avg_price=Avg('price'),
    min_price=Min('price'),
    max_price=Max('price'),
    total_value=Sum('price')
)
```

**SQL Equivalent:**

```sql
-- count()
SELECT COUNT(*) FROM users WHERE is_active = TRUE;

-- aggregate
SELECT
    COUNT(id) as total,
    AVG(age) as avg_age,
    MIN(age) as min_age,
    MAX(age) as max_age
FROM users;

-- Multiple aggregates
SELECT
    COUNT(id) as total_products,
    AVG(price) as avg_price,
    MIN(price) as min_price,
    MAX(price) as max_price,
    SUM(price) as total_value
FROM products;
```

### 6.2 Annotate (Add Fields to Each Row)

**Django ORM:**

```python
from django.db.models import Count, Sum, Avg

# Add count to each category
categories = Category.objects.annotate(
    product_count=Count('products')
)
for category in categories:
    print(f"{category.name}: {category.product_count} products")

# Add multiple annotations
categories = Category.objects.annotate(
    product_count=Count('products'),
    avg_price=Avg('products__price'),
    total_value=Sum('products__price')
)

# Filter after annotation
expensive_categories = Category.objects.annotate(
    avg_price=Avg('products__price')
).filter(avg_price__gt=100)

# Order by annotation
categories = Category.objects.annotate(
    product_count=Count('products')
).order_by('-product_count')
```

**SQL Equivalent:**

```sql
-- annotate with Count
SELECT
    c.*,
    COUNT(p.id) as product_count
FROM categories c
LEFT JOIN products p ON c.id = p.category_id
GROUP BY c.id;

-- Multiple annotations
SELECT
    c.*,
    COUNT(p.id) as product_count,
    AVG(p.price) as avg_price,
    SUM(p.price) as total_value
FROM categories c
LEFT JOIN products p ON c.id = p.category_id
GROUP BY c.id;

-- Filter after annotation (HAVING)
SELECT
    c.*,
    AVG(p.price) as avg_price
FROM categories c
LEFT JOIN products p ON c.id = p.category_id
GROUP BY c.id
HAVING AVG(p.price) > 100;
```

### 6.3 F Expressions (Database-Level Operations)

**Django ORM:**

```python
from django.db.models import F

# Reference field values
products = Product.objects.filter(price__gt=F('cost') * 1.5)

# Update using F expressions
Product.objects.all().update(price=F('price') * 1.1)  # 10% increase

# In annotations
products = Product.objects.annotate(
    profit=F('price') - F('cost'),
    profit_margin=(F('price') - F('cost')) / F('price') * 100
)

# Compare two fields
Product.objects.filter(price__gt=F('cost'))

# Span relationships
Product.objects.filter(price__gt=F('category__average_price'))
```

**SQL Equivalent:**

```sql
-- price > cost * 1.5
SELECT * FROM products WHERE price > cost * 1.5;

-- Update using field values
UPDATE products SET price = price * 1.1;

-- Annotations with calculations
SELECT
    *,
    (price - cost) as profit,
    ((price - cost) / price * 100) as profit_margin
FROM products;

-- Compare two fields
SELECT * FROM products WHERE price > cost;
```

### 6.4 Q Objects with Annotations

**Django ORM:**

```python
from django.db.models import Q, Count, Case, When, Value, CharField

# Conditional aggregation
categories = Category.objects.annotate(
    expensive_products=Count('products', filter=Q(products__price__gt=100)),
    cheap_products=Count('products', filter=Q(products__price__lte=100))
)

# Using Case/When
products = Product.objects.annotate(
    price_category=Case(
        When(price__lt=50, then=Value('Budget')),
        When(price__lt=200, then=Value('Mid-range')),
        default=Value('Premium'),
        output_field=CharField()
    )
)
```

**SQL Equivalent:**

```sql
-- Conditional aggregation
SELECT
    c.*,
    COUNT(p.id) FILTER (WHERE p.price > 100) as expensive_products,
    COUNT(p.id) FILTER (WHERE p.price <= 100) as cheap_products
FROM categories c
LEFT JOIN products p ON c.id = p.category_id
GROUP BY c.id;

-- Case/When
SELECT
    *,
    CASE
        WHEN price < 50 THEN 'Budget'
        WHEN price < 200 THEN 'Mid-range'
        ELSE 'Premium'
    END as price_category
FROM products;
```

---

## 7. Advanced Queries

### 7.1 Subqueries

**Django ORM:**

```python
from django.db.models import Subquery, OuterRef, Exists

# Subquery: Latest order for each user
latest_orders = Order.objects.filter(
    user=OuterRef('pk')
).order_by('-created_at')

users = User.objects.annotate(
    latest_order_date=Subquery(
        latest_orders.values('created_at')[:1]
    ),
    latest_order_total=Subquery(
        latest_orders.values('total_amount')[:1]
    )
)

# Exists
users_with_orders = User.objects.annotate(
    has_orders=Exists(Order.objects.filter(user=OuterRef('pk')))
).filter(has_orders=True)

# Filter using subquery
expensive_product_ids = Product.objects.filter(
    price__gt=1000
).values('id')

orders = Order.objects.filter(
    items__product_id__in=Subquery(expensive_product_ids)
).distinct()
```

**SQL Equivalent:**

```sql
-- Subquery for latest order
SELECT
    u.*,
    (SELECT o.created_at
     FROM orders o
     WHERE o.user_id = u.id
     ORDER BY o.created_at DESC
     LIMIT 1) as latest_order_date
FROM users u;

-- Exists
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Subquery in WHERE
SELECT DISTINCT o.*
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE oi.product_id IN (
    SELECT id FROM products WHERE price > 1000
);
```

### 7.2 Window Functions (Django 2.0+)

**Django ORM:**

```python
from django.db.models import Window, F
from django.db.models.functions import RowNumber, Rank, DenseRank

# Row number within category
products = Product.objects.annotate(
    row_number=Window(
        expression=RowNumber(),
        partition_by=[F('category')],
        order_by=F('price').desc()
    )
)

# Rank products by price
products = Product.objects.annotate(
    rank=Window(
        expression=Rank(),
        order_by=F('price').desc()
    )
)

# Dense rank within category
products = Product.objects.annotate(
    dense_rank=Window(
        expression=DenseRank(),
        partition_by=[F('category')],
        order_by=F('price').desc()
    )
)

# Running total
from django.db.models import Sum

orders = Order.objects.annotate(
    running_total=Window(
        expression=Sum('total_amount'),
        order_by=F('created_at').asc()
    )
)
```

**SQL Equivalent:**

```sql
-- Row number
SELECT
    *,
    ROW_NUMBER() OVER (
        PARTITION BY category_id
        ORDER BY price DESC
    ) as row_number
FROM products;

-- Rank
SELECT
    *,
    RANK() OVER (ORDER BY price DESC) as rank
FROM products;

-- Dense rank
SELECT
    *,
    DENSE_RANK() OVER (
        PARTITION BY category_id
        ORDER BY price DESC
    ) as dense_rank
FROM products;

-- Running total
SELECT
    *,
    SUM(total_amount) OVER (
        ORDER BY created_at
    ) as running_total
FROM orders;
```

### 7.3 Union, Intersection, Difference

**Django ORM:**

```python
# Union (combine querysets, remove duplicates)
young_users = User.objects.filter(age__lt=25)
gmail_users = User.objects.filter(email__endswith='@gmail.com')
combined = young_users.union(gmail_users)

# Union all (keep duplicates)
combined = young_users.union(gmail_users, all=True)

# Intersection (common records)
young_and_gmail = young_users.intersection(gmail_users)

# Difference (in first but not in second)
young_not_gmail = young_users.difference(gmail_users)

# Order after set operations
combined = young_users.union(gmail_users).order_by('username')
```

**SQL Equivalent:**

```sql
-- Union
SELECT * FROM users WHERE age < 25
UNION
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- Union all
SELECT * FROM users WHERE age < 25
UNION ALL
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- Intersection
SELECT * FROM users WHERE age < 25
INTERSECT
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- Difference
SELECT * FROM users WHERE age < 25
EXCEPT
SELECT * FROM users WHERE email LIKE '%@gmail.com';
```

### 7.4 Raw SQL (When Needed)

**Django ORM:**

```python
# Raw SQL queries
users = User.objects.raw('SELECT * FROM users WHERE age > %s', [25])

# With parameters (safe from SQL injection)
users = User.objects.raw(
    'SELECT * FROM users WHERE username = %s',
    ['maharab']
)

# Execute custom SQL
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute(
        "UPDATE users SET age = age + 1 WHERE is_active = %s",
        [True]
    )
    cursor.execute("SELECT * FROM users WHERE age > %s", [25])
    results = cursor.fetchall()
```

---

## 8. Transactions & ACID

### 8.1 ACID Properties

**ACID** stands for:

- **Atomicity**: All or nothing - either all operations succeed or all fail
- **Consistency**: Database remains in valid state before and after transaction
- **Isolation**: Transactions don't interfere with each other
- **Durability**: Committed changes are permanent

### 8.2 Basic Transactions

**Django ORM:**

```python
from django.db import transaction

# Method 1: Using decorator
@transaction.atomic
def create_user_with_profile(username, email):
    user = User.objects.create(username=username, email=email)
    Profile.objects.create(user=user, bio='New user')
    # If Profile creation fails, User creation is rolled back
    return user

# Method 2: Using context manager
def transfer_money(from_account, to_account, amount):
    with transaction.atomic():
        from_account.balance -= amount
        from_account.save()

        to_account.balance += amount
        to_account.save()

        # If any operation fails, all changes are rolled back

# Method 3: Manual transaction
from django.db import transaction

transaction.set_autocommit(False)
try:
    user = User.objects.create(username='test', email='test@example.com')
    Profile.objects.create(user=user)
    transaction.commit()
except Exception:
    transaction.rollback()
finally:
    transaction.set_autocommit(True)
```

**SQL Equivalent:**

```sql
-- Decorator or context manager
BEGIN;
    INSERT INTO users (username, email) VALUES ('maharab', 'maharab@gtaf.org');
    INSERT INTO profiles (user_id, bio) VALUES (1, 'New user');
COMMIT;
-- If error occurs:
-- ROLLBACK;

-- Transfer money
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### 8.3 Savepoints (Nested Transactions)

**Django ORM:**

```python
from django.db import transaction

def complex_operation():
    with transaction.atomic():
        user = User.objects.create(username='test', email='test@example.com')

        # Create a savepoint
        sid = transaction.savepoint()

        try:
            # Try something risky
            Profile.objects.create(user=user, bio='Test')
            transaction.savepoint_commit(sid)
        except Exception:
            # Rollback to savepoint (user creation still stands)
            transaction.savepoint_rollback(sid)
            # Create profile with default values
            Profile.objects.create(user=user, bio='Default')

        return user
```

**SQL Equivalent:**

```sql
BEGIN;
    INSERT INTO users (username, email) VALUES ('test', 'test@example.com');

    SAVEPOINT sp1;

    -- Try risky operation
    INSERT INTO profiles (user_id, bio) VALUES (1, 'Test');

    -- If it fails:
    ROLLBACK TO SAVEPOINT sp1;
    -- Then do alternative:
    INSERT INTO profiles (user_id, bio) VALUES (1, 'Default');

COMMIT;
```

### 8.4 Isolation Levels

**Django ORM:**

```python
from django.db import transaction

# Read committed (default in PostgreSQL)
with transaction.atomic():
    # Your code here
    pass

# Serializable (highest isolation)
from django.db import connection

with transaction.atomic():
    with connection.cursor() as cursor:
        cursor.execute('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE')

    # Your code here - no concurrent modifications allowed

# Read uncommitted (lowest isolation - rarely used)
with transaction.atomic():
    with connection.cursor() as cursor:
        cursor.execute('SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED')

    # Your code here
```

**SQL Equivalent:**

```sql
-- Set isolation level
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- Your queries
COMMIT;

-- Available levels:
-- READ UNCOMMITTED
-- READ COMMITTED (default)
-- REPEATABLE READ
-- SERIALIZABLE
```

### 8.5 Locking Rows (select_for_update)

**Django ORM:**

```python
from django.db import transaction

# Exclusive lock (others can't read or write)
with transaction.atomic():
    user = User.objects.select_for_update().get(id=1)
    user.balance += 100
    user.save()
    # Row is locked until transaction commits

# Share lock (others can read but not write)
with transaction.atomic():
    users = User.objects.select_for_update(of=('self',)).filter(
        is_active=True
    )
    # Process users safely

# Skip locked rows (don't wait)
with transaction.atomic():
    user = User.objects.select_for_update(skip_locked=True).first()
    if user:
        # Process this user
        pass
    # If all rows are locked, user will be None

# No wait (raise error if locked)
from django.db.utils import DatabaseError

with transaction.atomic():
    try:
        user = User.objects.select_for_update(nowait=True).get(id=1)
        # Process user
    except DatabaseError:
        # Row is locked, handle accordingly
        pass
```

**SQL Equivalent:**

```sql
-- select_for_update()
BEGIN;
    SELECT * FROM users WHERE id = 1 FOR UPDATE;
    UPDATE users SET balance = balance + 100 WHERE id = 1;
COMMIT;

-- select_for_update(of=('self',))
BEGIN;
    SELECT * FROM users WHERE is_active = TRUE FOR UPDATE OF users;
COMMIT;

-- skip_locked
BEGIN;
    SELECT * FROM users WHERE is_active = TRUE
    FOR UPDATE SKIP LOCKED
    LIMIT 1;
COMMIT;

-- nowait
BEGIN;
    SELECT * FROM users WHERE id = 1 FOR UPDATE NOWAIT;
COMMIT;
```

### 8.6 Real-World Transaction Patterns

#### Pattern 1: Bank Transfer (ACID Example)

**Django ORM:**

```python
from django.db import transaction
from decimal import Decimal

@transaction.atomic
def transfer_funds(from_account_id, to_account_id, amount):
    """Transfer money between accounts atomically"""
    # Lock both accounts to prevent race conditions
    from_account = Account.objects.select_for_update().get(id=from_account_id)
    to_account = Account.objects.select_for_update().get(id=to_account_id)

    # Validate balance
    if from_account.balance < amount:
        raise ValueError("Insufficient funds")

    # Perform transfer
    from_account.balance -= amount
    from_account.save(update_fields=['balance'])

    to_account.balance += amount
    to_account.save(update_fields=['balance'])

    # Log transaction
    Transaction.objects.create(
        from_account=from_account,
        to_account=to_account,
        amount=amount,
        status='completed'
    )

    return True
```

#### Pattern 2: Inventory Management

**Django ORM:**

```python
@transaction.atomic
def process_order(order_id):
    """Process order and update inventory atomically"""
    order = Order.objects.select_for_update().get(id=order_id)

    if order.status != 'pending':
        raise ValueError("Order already processed")

    # Lock and update inventory
    for item in order.items.all():
        product = Product.objects.select_for_update().get(id=item.product_id)

        if product.stock < item.quantity:
            raise ValueError(f"Insufficient stock for {product.name}")

        product.stock -= item.quantity
        product.save(update_fields=['stock'])

    # Update order status
    order.status = 'processing'
    order.save(update_fields=['status'])

    return order
```

#### Pattern 3: Idempotent Operations

**Django ORM:**

```python
@transaction.atomic
def create_payment(order_id, amount, payment_id):
    """Create payment idempotently (safe to retry)"""
    # Check if payment already exists
    payment, created = Payment.objects.get_or_create(
        payment_id=payment_id,  # External payment ID
        defaults={
            'order_id': order_id,
            'amount': amount,
            'status': 'pending'
        }
    )

    if created:
        # First time processing this payment
        order = Order.objects.select_for_update().get(id=order_id)
        order.paid_amount += amount
        order.save(update_fields=['paid_amount'])

        payment.status = 'completed'
        payment.save(update_fields=['status'])

    return payment, created
```

---

## 9. Performance Optimization

### 9.1 Query Optimization Checklist

```python
# ❌ BAD: N+1 queries
products = Product.objects.all()
for product in products:
    print(product.category.name)  # Extra query per product

# ✅ GOOD: Use select_related
products = Product.objects.select_related('category').all()
for product in products:
    print(product.category.name)  # No extra queries

# ❌ BAD: Loading unnecessary data
users = User.objects.all()  # Loads all fields

# ✅ GOOD: Load only what you need
users = User.objects.only('id', 'username', 'email')
users = User.objects.values('id', 'username')  # Even lighter

# ❌ BAD: Multiple queries in loop
for user_id in user_ids:
    user = User.objects.get(id=user_id)
    process(user)

# ✅ GOOD: Fetch all at once
users = User.objects.filter(id__in=user_ids)
user_dict = {user.id: user for user in users}
for user_id in user_ids:
    process(user_dict[user_id])

# ❌ BAD: Count with len()
users = User.objects.all()
count = len(users)  # Loads all users into memory!

# ✅ GOOD: Use count()
count = User.objects.count()  # Database-level count

# ❌ BAD: Checking existence with len() or count()
if User.objects.filter(email='test@example.com').count() > 0:
    pass

# ✅ GOOD: Use exists()
if User.objects.filter(email='test@example.com').exists():
    pass
```

### 9.2 Bulk Operations

**Django ORM:**

```python
# ❌ BAD: Individual creates (N queries)
for i in range(1000):
    User.objects.create(username=f'user{i}', email=f'user{i}@example.com')

# ✅ GOOD: Bulk create (1 query)
users = [
    User(username=f'user{i}', email=f'user{i}@example.com')
    for i in range(1000)
]
User.objects.bulk_create(users, batch_size=500)

# ❌ BAD: Individual updates (N queries)
users = User.objects.all()
for user in users:
    user.is_active = True
    user.save()

# ✅ GOOD: Bulk update (1 query)
User.objects.all().update(is_active=True)

# ✅ GOOD: Bulk update specific fields
users = User.objects.filter(age__lt=18)
for user in users:
    user.is_active = False
    user.status = 'minor'
User.objects.bulk_update(users, ['is_active', 'status'], batch_size=500)

# ❌ BAD: Individual deletes (N queries)
users = User.objects.filter(is_active=False)
for user in users:
    user.delete()

# ✅ GOOD: Bulk delete (1 query)
User.objects.filter(is_active=False).delete()
```

### 9.3 Database Functions

**Django ORM:**

```python
from django.db.models.functions import (
    Lower, Upper, Concat, Substr, Length,
    Coalesce, Cast, Now, ExtractYear
)
from django.db.models import Value, CharField

# String functions
users = User.objects.annotate(
    lower_username=Lower('username'),
    full_name=Concat('first_name', Value(' '), 'last_name')
)

# Date functions
from django.db.models.functions import TruncDate
orders = Order.objects.annotate(
    year=ExtractYear('created_at'),
    order_date=TruncDate('created_at')
)

# Coalesce (return first non-null value)
products = Product.objects.annotate(
    display_price=Coalesce('sale_price', 'price')
)

# Cast types
users = User.objects.annotate(
    age_str=Cast('age', CharField())
)
```

### 9.4 Caching Query Results

**Django ORM:**

```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

# Manual caching
def get_featured_products():
    cache_key = 'featured_products'
    products = cache.get(cache_key)

    if products is None:
        products = list(
            Product.objects.filter(is_featured=True)
            .select_related('category')
            [:10]
        )
        cache.set(cache_key, products, 60 * 15)  # Cache for 15 minutes

    return products

# View caching
@cache_page(60 * 15)  # Cache for 15 minutes
def product_list(request):
    products = Product.objects.all()
    return render(request, 'products.html', {'products': products})

# Cache invalidation
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete('featured_products')
    cache.delete(f'product_{instance.id}')
```

### 9.5 Iterator() for Large Querysets

**Django ORM:**

```python
# ❌ BAD: Loads all records into memory
users = User.objects.all()
for user in users:
    process(user)  # If you have millions, you'll run out of memory

# ✅ GOOD: Use iterator() for large datasets
for user in User.objects.all().iterator(chunk_size=2000):
    process(user)  # Fetches in chunks, memory-efficient

# ✅ GOOD: Use iterator with select_related
for product in Product.objects.select_related('category').iterator(chunk_size=1000):
    process(product)
```

### 9.6 only() vs defer() vs values()

**Django ORM:**

```python
# only() - Load only specified fields (returns model instances)
users = User.objects.only('username', 'email')
# Can access username and email without extra queries
# Accessing other fields triggers additional queries

# defer() - Load all except specified fields (returns model instances)
users = User.objects.defer('bio', 'avatar')
# All fields except bio and avatar are loaded
# Accessing deferred fields triggers additional queries

# values() - Returns dictionaries, not model instances
users = User.objects.values('username', 'email')
# Returns: [{'username': 'maharab', 'email': '...'}, ...]
# Lightweight, but no model methods/properties

# values_list() - Returns tuples
users = User.objects.values_list('username', 'email')
# Returns: [('maharab', '...'), ...]

# When to use what:
# - only(): Need model instances, accessing few fields
# - defer(): Need model instances, excluding few fields
# - values(): Don't need model instances, want dictionaries
# - values_list(): Don't need model instances, want tuples
```

---

## 10. Database Indexing in Django

### 10.1 Index Types in Models

**Django Models:**

```python
class Product(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)  # Automatically indexed (unique)
    category = models.ForeignKey(Category, on_delete=models.CASCADE)  # Automatically indexed
    price = models.DecimalField(max_digits=10, decimal_places=2)
    is_active = models.BooleanField(default=True, db_index=True)  # Add index
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # Simple index
            models.Index(fields=['price']),

            # Composite index
            models.Index(fields=['category', 'price']),

            # Descending index
            models.Index(fields=['-created_at']),

            # Partial index (PostgreSQL)
            models.Index(
                fields=['price'],
                name='active_product_price_idx',
                condition=Q(is_active=True)
            ),

            # Named index
            models.Index(fields=['name'], name='product_name_idx'),
        ]
```

**SQL Equivalent:**

```sql
-- Simple index
CREATE INDEX product_price_idx ON products(price);

-- Composite index
CREATE INDEX product_category_price_idx ON products(category_id, price);

-- Descending index
CREATE INDEX product_created_desc_idx ON products(created_at DESC);

-- Partial index
CREATE INDEX active_product_price_idx ON products(price)
WHERE is_active = TRUE;

-- Named index
CREATE INDEX product_name_idx ON products(name);
```

### 10.2 When to Add Indexes

```python
# ✅ Index these:
# 1. Foreign keys (Django does this automatically)
# 2. Fields used in WHERE clauses frequently
# 3. Fields used in ORDER BY
# 4. Fields used in joins
# 5. Fields with high cardinality (many unique values)

# ❌ Don't index these:
# 1. Small tables (< 1000 rows)
# 2. Fields with low cardinality (few unique values like boolean)
# 3. Fields rarely queried
# 4. Fields in tables with frequent writes (indexes slow down writes)
```

### 10.3 Full-Text Search Indexes (PostgreSQL)

**Django Models:**

```python
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.search import SearchVectorField

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    search_vector = SearchVectorField(null=True)

    class Meta:
        indexes = [
            GinIndex(fields=['search_vector']),
        ]

# Update search vector
from django.contrib.postgres.search import SearchVector

Article.objects.update(search_vector=SearchVector('title', 'content'))

# Search
from django.contrib.postgres.search import SearchQuery

query = SearchQuery('django orm')
articles = Article.objects.filter(search_vector=query)
```

---

## 11. Real-World Patterns (GTAF Context)

### 11.1 Quran Verses with Translations

**Models:**

```python
class Surah(models.Model):
    number = models.IntegerField(unique=True)
    name_arabic = models.CharField(max_length=100)
    name_english = models.CharField(max_length=100)
    revelation_place = models.CharField(max_length=10)

    class Meta:
        ordering = ['number']

class Verse(models.Model):
    surah = models.ForeignKey(Surah, on_delete=models.CASCADE, related_name='verses')
    verse_number = models.IntegerField()
    text_arabic = models.TextField()

    class Meta:
        unique_together = ('surah', 'verse_number')
        ordering = ['surah', 'verse_number']

class Translation(models.Model):
    verse = models.ForeignKey(Verse, on_delete=models.CASCADE, related_name='translations')
    language = models.CharField(max_length=10)  # 'en', 'bn', 'ur'
    translator = models.CharField(max_length=100)
    text = models.TextField()

    class Meta:
        unique_together = ('verse', 'language', 'translator')
```

**Optimized Queries:**

```python
from django.db.models import Prefetch

# Get verses with multiple translations efficiently
verses = Verse.objects.filter(
    surah__number=1
).select_related('surah').prefetch_related(
    Prefetch(
        'translations',
        queryset=Translation.objects.filter(
            language__in=['en', 'bn']
        ).order_by('language')
    )
)

# Search in translations
verses_with_paradise = Verse.objects.filter(
    translations__text__icontains='paradise',
    translations__language='en'
).distinct()
```

### 11.2 Donation Platform with Transactions

**Models:**

```python
class Campaign(models.Model):
    title = models.CharField(max_length=200)
    goal_amount = models.DecimalField(max_digits=12, decimal_places=2)
    raised_amount = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    is_active = models.BooleanField(default=True)

class Donation(models.Model):
    PAYMENT_STATUS = (
        ('pending', 'Pending'),
        ('completed', 'Completed'),
        ('failed', 'Failed'),
        ('refunded', 'Refunded'),
    )

    campaign = models.ForeignKey(Campaign, on_delete=models.CASCADE, related_name='donations')
    donor_name = models.CharField(max_length=100)
    donor_email = models.EmailField()
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_status = models.CharField(max_length=20, choices=PAYMENT_STATUS, default='pending')
    payment_id = models.CharField(max_length=100, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    completed_at = models.DateTimeField(null=True, blank=True)
```

**Optimized Queries with Transactions:**

```python
from django.db import transaction
from django.db.models import Sum, Count, F, Q
from django.utils import timezone

@transaction.atomic
def complete_donation(payment_id, transaction_id):
    """Complete donation and update campaign atomically"""
    # Lock donation and campaign
    donation = Donation.objects.select_for_update().get(
        payment_id=payment_id,
        payment_status='pending'
    )

    campaign = Campaign.objects.select_for_update().get(
        id=donation.campaign_id
    )

    # Update donation
    donation.payment_status = 'completed'
    donation.completed_at = timezone.now()
    donation.save(update_fields=['payment_status', 'completed_at'])

    # Update campaign
    campaign.raised_amount = F('raised_amount') + donation.amount
    campaign.save(update_fields=['raised_amount'])

    return donation

# Get campaign statistics
campaigns = Campaign.objects.filter(is_active=True).annotate(
    total_donations=Count('donations', filter=Q(donations__payment_status='completed')),
    total_raised=Sum('donations__amount', filter=Q(donations__payment_status='completed'))
).order_by('-total_raised')
```

---

## 12. Common Pitfalls & Solutions

### 12.1 Pitfall: Evaluating QuerySets Multiple Times

```python
# ❌ BAD: QuerySet evaluated multiple times
users = User.objects.filter(is_active=True)
count = len(users)  # Evaluation 1
first = users[0]     # Evaluation 2
for user in users:   # Evaluation 3
    print(user)

# ✅ GOOD: Evaluate once
users = list(User.objects.filter(is_active=True))
count = len(users)
first = users[0]
for user in users:
    print(user)
```

### 12.2 Pitfall: Using len() Instead of count()

```python
# ❌ BAD: Loads all records into memory
user_count = len(User.objects.all())

# ✅ GOOD: Database-level count
user_count = User.objects.count()
```

### 12.3 Pitfall: Not Using Transactions

```python
# ❌ BAD: Not atomic
def transfer_money(from_user, to_user, amount):
    from_user.balance -= amount
    from_user.save()
    # If this fails, from_user lost money but to_user didn't receive!
    to_user.balance += amount
    to_user.save()

# ✅ GOOD: Use transaction
@transaction.atomic
def transfer_money(from_user, to_user, amount):
    from_user.balance -= amount
    from_user.save()
    to_user.balance += amount
    to_user.save()
```

---

## 13. Query Debugging & Monitoring

### 13.1 View Generated SQL

```python
# Method 1: Using .query attribute
queryset = User.objects.filter(is_active=True)
print(queryset.query)

# Method 2: Pretty print SQL
import sqlparse
print(sqlparse.format(str(queryset.query), reindent=True, keyword_case='upper'))
```

### 13.2 Django Debug Toolbar

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = ['127.0.0.1']

# urls.py
from django.urls import include, path

urlpatterns = [
    # ...
    path('__debug__/', include('debug_toolbar.urls')),
]
```

### 13.3 Custom Query Monitoring

```python
from django.db import connection
from django.test.utils import CaptureQueriesContext

def profile_queries(func):
    """Decorator to profile database queries"""
    def wrapper(*args, **kwargs):
        with CaptureQueriesContext(connection) as queries:
            result = func(*args, **kwargs)

        print(f"\n{func.__name__} executed {len(queries)} queries:")
        for i, query in enumerate(queries, 1):
            print(f"\nQuery {i}: {query['sql']}")
            print(f"Time: {query['time']}s")

        return result
    return wrapper

# Usage
@profile_queries
def get_products():
    products = Product.objects.select_related('category').all()[:10]
    return list(products)
```

---

## Summary & Quick Reference

### Django ORM to SQL Mapping

| Task         | Django ORM                          | SQL                              |
| ------------ | ----------------------------------- | -------------------------------- |
| Get all      | `Model.objects.all()`               | `SELECT * FROM table`            |
| Filter       | `Model.objects.filter(field=value)` | `SELECT * WHERE field = value`   |
| Get one      | `Model.objects.get(id=1)`           | `SELECT * WHERE id = 1 LIMIT 1`  |
| Order        | `.order_by('field')`                | `ORDER BY field`                 |
| Limit        | `[:10]`                             | `LIMIT 10`                       |
| Count        | `.count()`                          | `SELECT COUNT(*)`                |
| Update       | `.update(field=value)`              | `UPDATE table SET field = value` |
| Delete       | `.delete()`                         | `DELETE FROM table`              |
| Join         | `.select_related('fk')`             | `INNER JOIN`                     |
| Reverse Join | `.prefetch_related('reverse')`      | `SELECT ... WHERE fk IN (...)`   |
| Aggregate    | `.aggregate(Avg('field'))`          | `SELECT AVG(field)`              |
| Annotate     | `.annotate(count=Count('rel'))`     | `SELECT ... COUNT(...) GROUP BY` |

### Performance Best Practices

✅ **Always do:**

- Use `select_related()` for ForeignKey/OneToOne
- Use `prefetch_related()` for reverse ForeignKey/ManyToMany
- Use `only()` or `values()` when you don't need full objects
- Use `bulk_create()` / `bulk_update()` for multiple records
- Use `exists()` instead of `count()` or `len()`
- Use `iterator()` for large querysets
- Add indexes to frequently queried fields
- Use transactions for data integrity
- Use `F()` expressions for database-level operations

❌ **Never do:**

- Access relationships in loops without prefetching (N+1 problem)
- Use `len()` on querysets (use `count()`)
- Evaluate querysets multiple times
- Skip transactions for critical operations

---

**Created for Maharab - Backend Engineer at Greentech Apps Foundation**
_Master Django ORM with SQL Understanding and Performance Optimization_
