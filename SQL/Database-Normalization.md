# Database Normalization — Interview Revision Notes

**Normalization** is the process of organizing data into related tables to **reduce redundancy** and prevent **insert, update, and delete anomalies**.

Think of it as:

> **1NF → Atomic data**
> **2NF → No partial dependency**
> **3NF → No transitive dependency**

---

## 1️⃣ First Normal Form — 1NF

### Rule

A table is in **1NF** when:

* Each cell contains **one atomic value**
* No comma-separated/repeating values in a column
* Each row can be uniquely identified

### ❌ Before 1NF

| student_id | student_name | course           |
| ---------: | ------------ | ---------------- |
|          1 | John         | Math, Science    |
|          2 | Sarah        | English, History |
|          3 | Mike         | Math, Art        |

Problem: `course` contains **multiple values in one cell**.

### ✅ After 1NF

| student_id | student_name | course  |
| ---------: | ------------ | ------- |
|          1 | John         | Math    |
|          1 | John         | Science |
|          2 | Sarah        | English |
|          2 | Sarah        | History |
|          3 | Mike         | Math    |
|          3 | Mike         | Art     |

Now every cell contains **exactly one value**.

**Interview shortcut:**

> **1NF = Atomic values / no repeating groups.**

---

# 2️⃣ Second Normal Form — 2NF

### Rule

For a table to be in **2NF**:

1. It must already be in **1NF**
2. Every non-key column must depend on the **entire primary key**
3. There should be **no partial dependency**

This matters mainly when we have a **composite primary key**.

### ❌ Before 2NF

Suppose:

**Primary Key = `(student_id, course_id)`**

| student_id | course_id | student_name | grade |
| ---------: | --------- | ------------ | ----- |
|          1 | C101      | John         | A     |
|          1 | C102      | John         | B     |
|          2 | C101      | Sarah        | A     |
|          2 | C103      | Sarah        | B     |

Look at the dependencies:

```text
(student_id, course_id) → grade

student_id → student_name
```

`grade` depends on the **complete composite key**, which is correct.

But:

```text
student_name
     ↑
 student_id
```

`student_name` depends only on **part of the primary key** (`student_id`).

That's a **partial dependency**.

### ✅ After 2NF

Split it into two tables.

**Students**

| student_id | student_name |
| ---------: | ------------ |
|          1 | John         |
|          2 | Sarah        |

**Enrollments**

| student_id | course_id | grade |
| ---------: | --------- | ----- |
|          1 | C101      | A     |
|          1 | C102      | B     |
|          2 | C101      | A     |
|          2 | C103      | B     |

Now:

```text
Students
student_id → student_name

Enrollments
(student_id, course_id) → grade
```

No partial dependency remains.

**Interview shortcut:**

> **2NF = 1NF + no partial dependency.**

Or remember:

> **Non-key attributes must depend on the WHOLE key.**

---

# 3️⃣ Third Normal Form — 3NF

### Rule

For a table to be in **3NF**:

1. It must already be in **2NF**
2. There should be **no transitive dependency**
3. A non-key column should not depend on another non-key column

### ❌ Before 3NF

| employee_id | emp_name | dept_id | dept_name |
| ----------: | -------- | ------- | --------- |
|           1 | John     | D1      | IT        |
|           2 | Sarah    | D1      | IT        |
|           3 | Mike     | D2      | HR        |

Primary key:

```text
employee_id
```

Dependencies:

```text
employee_id → emp_name
employee_id → dept_id

dept_id → dept_name
```

Therefore:

```text
employee_id
     ↓
  dept_id
     ↓
 dept_name
```

So indirectly:

```text
employee_id → dept_id → dept_name
```

This is a **transitive dependency**.

`dept_name` depends on `dept_id`, which is another **non-key attribute**.

### ✅ After 3NF

Split into:

**Employees**

| employee_id | emp_name | dept_id |
| ----------: | -------- | ------- |
|           1 | John     | D1      |
|           2 | Sarah    | D1      |
|           3 | Mike     | D2      |

**Departments**

| dept_id | dept_name |
| ------- | --------- |
| D1      | IT        |
| D2      | HR        |

Now:

```text
employee_id → emp_name
employee_id → dept_id

dept_id → dept_name
```

Each fact is stored in the appropriate table.

**Interview shortcut:**

> **3NF = 2NF + no transitive dependency.**

Or:

> **Non-key attributes should depend only on the key, not another non-key attribute.**

---

# 🧠 Best Way to Remember 1NF → 2NF → 3NF

```text
                 NORMALIZATION
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
       1NF            2NF            3NF
        │              │              │
     Atomic         No Partial     No Transitive
     Values         Dependency      Dependency
        │              │              │
        ▼              ▼              ▼
 Math,Science    Composite Key     Emp → Dept
      ❌           Problem          → DeptName
        │              │              │
        ▼              ▼              ▼
 Math              Student       Separate
 Science           + Enrollment   Department
```

A very useful sentence for interviews is:

> **“The key, the whole key, and nothing but the key.”**

Think:

**1NF:** values are atomic.
**2NF:** depend on the **whole key**.
**3NF:** depend on **nothing but the key**.

---

## Why do we normalize?

Normalization primarily helps avoid three anomalies.

**Update anomaly:** If `IT` department appears for 1,000 employees and its name changes, we shouldn't update 1,000 rows.

**Insert anomaly:** We should be able to create a department even when it currently has no employee.

**Delete anomaly:** Deleting the last employee in HR should not accidentally delete the information that the HR department exists.

---

## 🎯 30-second interview answer

> **“Normalization organizes relational data to reduce redundancy and prevent insert, update, and delete anomalies. In 1NF, each column contains atomic values. In 2NF, the table is in 1NF and every non-key attribute depends on the complete primary key, so there is no partial dependency. In 3NF, the table is in 2NF and there is no transitive dependency, meaning a non-key attribute should not depend on another non-key attribute. For example, Employee → DepartmentId → DepartmentName violates 3NF, so Department should be moved to a separate table.”**





-------------------###----------------------

Database Normalization is the process of organizing tables to eliminate duplicate data and prevent update, insert, and delete errors.
Let’s walk through 1NF, 2NF, and 3NF step by step by fixing a messy, unnormalized Employee_Projects table.

Starting Point: Unnormalized Table (0NF)


emp_id	emp_name	dept_id	dept_name	project_ids
1	Neha	D1	Executive	P101, P102
2	Asha	D2	Engineering	P101

Problems here: Multiple values in one cell (P101, P102), repeated department names, and awkward updates.

1. First Normal Form (1NF) — Atomic Values & Unique Rows

Rule:
• Every cell must hold only one single (atomic) value (no comma-separated lists).
• Each row must be uniquely identifiable (using a Primary Key).

How We Fix It:

Split Neha's projects into separate rows and define (emp_id, project_id) as the composite Primary Key.
Employee_Projects_1NF

emp_id (PK)	project_id (PK)	emp_name	dept_id	dept_name
1	P101	Neha	D1	Executive
1	P102	Neha	D1	Executive
2	P101	Asha	D2	Engineering

(1NF is achieved, but notice emp_name, dept_id, and dept_name are still repeated every time Neha joins a new project.)

2. Second Normal Form (2NF) — No Partial Dependencies

Rule:
• Must already be in 1NF.
• Every non-key column must depend on the entire Primary Key, not just part of it. (Applies when you have a composite Primary Key).

What's Wrong in 1NF?

Our Primary Key is (emp_id, project_id). However, emp_name, dept_id, and dept_name depend only on emp_id—they have nothing to do with project_id. This is a Partial Dependency.

How We Fix It:

Move employee details into their own Employees table, leaving only the mapping in Employee_Projects.
Table 1: Employees_2NF

emp_id (PK)	emp_name	dept_id	dept_name
1	Neha	D1	Executive
2	Asha	D2	Engineering

Table 2: Employee_Projects

emp_id (PK, FK)	project_id (PK)
1	P101
1	P102
2	P101

(2NF is achieved, but notice dept_name is still repeated for every employee in the same department.)

3. Third Normal Form (3NF) — No Transitive Dependencies

Rule:
• Must already be in 2NF.
• Non-key columns must depend only on the Primary Key, not on another non-key column (no Transitive Dependencies).

What's Wrong in 2NF?

In Employees_2NF, emp_id → dept_id → dept_name.
dept_name depends on dept_id (a non-key column), rather than directly on emp_id. If the "Engineering" department is renamed to "Product Engineering", you would have to update multiple employee rows.

How We Fix It:

Move department details into a separate Departments table and link it via dept_id.
Table 1: Departments
sql
CREATE TABLE Departments (
    dept_id VARCHAR(10) PRIMARY KEY,
    dept_name VARCHAR(50)
);
Use code with caution.

dept_id (PK)	dept_name
D1	Executive
D2	Engineering

Table 2: Employees
sql
CREATE TABLE Employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(50),
    dept_id VARCHAR(10) REFERENCES Departments(dept_id)
);
Use code with caution.

emp_id (PK)	emp_name	dept_id (FK)
1	Neha	D1
2	Asha	D2

Table 3: Employee_Projects
sql
CREATE TABLE Employee_Projects (
    emp_id INT REFERENCES Employees(emp_id),
    project_id VARCHAR(10),
    PRIMARY KEY (emp_id, project_id)
);
Use code with caution.

emp_id (PK, FK)	project_id (PK)
1	P101
1	P102
2	P101


Easy Memory Trick

Every non-key column must provide a fact about:
1. The Key (1NF)
2. The Whole Key (2NF)
3. Nothing But the Key (3NF) — so help me Codd!
If you'd like, I can also show BCNF (Boyce-Codd Normal Form), when to intentionally break normalization (Denormalization) for faster reporting queries, or how to query all three 3NF tables back together using JOIN.
