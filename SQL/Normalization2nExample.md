# **Database Normalization – 1NF, 2NF, and 3NF with Examples**

## **1. What is Database Normalization?**

**Database Normalization** is the process of organizing database tables to reduce data redundancy, eliminate duplicate data, and prevent data anomalies.

It helps prevent three common problems:

1. **Insert Anomaly:** Difficulty inserting data without unrelated data.
2. **Update Anomaly:** Updating the same information in multiple rows.
3. **Delete Anomaly:** Accidentally losing important information when deleting a record.

Let's understand the three most commonly used Normal Forms (1NF, 2NF, and 3NF) step by step using an Employee-Projects example.

## **2. Starting Point – Unnormalized Table (0NF)**

Consider the following table:

**Employee_Projects (Unnormalized)**

| emp_id | emp_name | dept_id | dept_name   | project_ids |
| ------ | -------- | ------- | ----------- | ----------- |
| 1      | Neha     | D1      | Executive   | P101, P102  |
| 2      | Asha     | D2      | Engineering | P101        |

### **Problems**

- Multiple project IDs are stored in a single column.
- Department names are repeated.
- Updating employee or department information can become difficult.
- Searching and maintaining individual project assignments is complicated.

We can solve these problems through normalization.

## **3. First Normal Form (1NF) – Atomic Values & Unique Rows**

### **Rule**

A table is in 1NF when:

1. Every column contains atomic (single) values.
2. There are no multiple values or comma-separated lists in a cell.
3. Each row can be uniquely identified using a Primary Key.

### **How to achieve 1NF?**

Split multiple project IDs into separate rows.

**Employee_Projects_1NF**

| emp_id (PK) | project_id (PK) | emp_name | dept_id | dept_name   |
| ----------- | --------------- | -------- | ------- | ----------- |
| 1           | P101            | Neha     | D1      | Executive   |
| 1           | P102            | Neha     | D1      | Executive   |
| 2           | P101            | Asha     | D2      | Engineering |

**Primary Key:** Composite Primary Key `(emp_id, project_id)`.

### **What did we achieve?**

- Each cell contains a single value.
- Each employee-project combination is uniquely identifiable.
- Multiple projects are represented as separate records.

### **Remaining Problem**

Employee information is repeated whenever an employee joins another project.

For example, Neha's name and department are repeated for P101 and P102.

This leads us to 2NF.

## **4. Second Normal Form (2NF) – No Partial Dependencies**

### **Rule**

A table is in 2NF when:

1. It is already in 1NF.
2. Every non-key attribute depends on the entire Primary Key, not just part of it.

**Important:** Partial dependency is relevant when a candidate key consists of multiple columns.

### **What's wrong with 1NF?**

Our composite Primary Key is:

`(emp_id, project_id)`

However:

- `emp_name` depends only on `emp_id`.
- `dept_id` depends only on `emp_id`.
- `dept_name` depends only on `emp_id` in this example.

They do not depend on the complete composite key.

This is called a **Partial Dependency**.

### **How to achieve 2NF?**

Separate employee information from employee-project mapping.

**Table 1: Employees_2NF**

| emp_id (PK) | emp_name | dept_id | dept_name   |
| ----------- | -------- | ------- | ----------- |
| 1           | Neha     | D1      | Executive   |
| 2           | Asha     | D2      | Engineering |

**Table 2: Employee_Projects**

| emp_id (PK, FK) | project_id (PK) |
| --------------- | --------------- |
| 1               | P101            |
| 1               | P102            |
| 2               | P101            |

### **What did we achieve?**

- Employee details are maintained separately.
- Employee-project relationships are maintained independently.
- Partial dependencies are eliminated from the mapping table.
- Employee details are no longer repeated for every project.

### **Remaining Problem**

Department information is still stored in the Employees table.

For example, if 100 employees belong to the Engineering department, the department name is stored 100 times.

If the department name changes, multiple employee records need updating.

This leads us to 3NF.

## **5. Third Normal Form (3NF) – No Transitive Dependencies**

### **Rule**

A table is in 3NF when:

1. It is already in 2NF.
2. Non-key attributes must depend directly on the Primary Key, not on another non-key attribute.

### **What's wrong with 2NF?**

Consider the following functional dependencies:

```
emp_id → dept_id
dept_id → dept_name

Therefore:

emp_id → dept_id → dept_name
```

Here:

- `emp_id` is the Primary Key.
- `dept_id` determines `dept_name`.
- `dept_name` depends on another non-key attribute (`dept_id`).

This is called a **Transitive Dependency**.

### **How to achieve 3NF?**

Move department details into a separate Departments table.

We now have three tables.

**Table 1: Departments**

```
CREATE TABLE Departments
(
    dept_id VARCHAR(10) PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL
);
```

| dept_id (PK) | dept_name   |
| ------------ | ----------- |
| D1           | Executive   |
| D2           | Engineering |

**Table 2: Employees**

```
CREATE TABLE Employees
(
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(50) NOT NULL,
    dept_id VARCHAR(10),
    FOREIGN KEY (dept_id) REFERENCES Departments(dept_id)
);
```

| emp_id (PK) | emp_name | dept_id (FK) |
| ----------- | -------- | ------------ |
| 1           | Neha     | D1           |
| 2           | Asha     | D2           |

**Table 3: Employee_Projects**

```
CREATE TABLE Employee_Projects
(
    emp_id INT,
    project_id VARCHAR(10),
    PRIMARY KEY (emp_id, project_id),
    FOREIGN KEY (emp_id) REFERENCES Employees(emp_id)
);
```

| emp_id (PK, FK) | project_id (PK) |
| --------------- | --------------- |
| 1               | P101            |
| 1               | P102            |
| 2               | P101            |

### **What did we achieve?**

- Department information is maintained in one place.
- Employee information is maintained separately.
- Project assignments are maintained independently.
- Partial and transitive dependencies are eliminated.
- Data redundancy is reduced.

Final normalized design – 3NF

**Departments**

PK: dept_id

dept_name

One department → Many employees

**Employees**

PK: emp_id | FK: dept_id

emp_name

One employee → Many project assignments

**Employee_Projects**

Composite PK: emp_id + project_id

FK: emp_id

## **6. Compare 1NF vs 2NF vs 3NF**

| Normal Form | Main Rule                | Problem Solved                                           |
| ----------- | ------------------------ | -------------------------------------------------------- |
| 1NF         | Atomic values            | Multiple values in one cell                              |
| 2NF         | No partial dependency    | Dependency on part of composite key                      |
| 3NF         | No transitive dependency | Non-key attribute depending on another non-key attribute |

## **7. Easy Memory Trick for Interviews**

Remember this famous normalization phrase:

**Every non-key attribute must depend on:**

1. **1NF – The Key:** Each cell contains one atomic value.
2. **2NF – The Whole Key:** Every non-key attribute depends on the entire candidate key.
3. **3NF – Nothing But the Key:** No transitive dependency.

"Every non-key attribute must depend on the key, the whole key, and nothing but the key."

## **8. Interview Questions and Answers**

**Q1. What is the difference between 2NF and 3NF?**

- 2NF eliminates partial dependencies.
- 3NF eliminates transitive dependencies.

**Q2. Can a table be in 1NF but not in 2NF?**

Yes. A table can have atomic values but still contain partial dependencies on a composite Primary Key.

**Q3. Can a table be in 2NF but not in 3NF?**

Yes. A table can have no partial dependencies but still contain transitive dependencies.

**Q4. Is normalization always good for performance?**

Not necessarily. Normalization reduces data duplication and improves data consistency, but queries may require more JOIN operations.

For read-heavy reporting systems, denormalization may improve query performance.

**Q5. What is denormalization?**

Denormalization is the process of intentionally introducing some data redundancy to reduce JOIN operations and improve read performance.

**Q6. What is BCNF?**

Boyce-Codd Normal Form (BCNF) is a stricter version of 3NF. For every non-trivial functional dependency `X → Y`, X must be a super key.

## **9. Practical SQL – Fetch Employee, Department and Project Details**

After normalization, we can retrieve all related information using JOINs.

```
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    ep.project_id
FROM Employees e
INNER JOIN Departments d
    ON e.dept_id = d.dept_id
INNER JOIN Employee_Projects ep
    ON e.emp_id = ep.emp_id
ORDER BY e.emp_id;
```

**Expected Output:**

| emp_id | emp_name | dept_name   | project_id |
| ------ | -------- | ----------- | ---------- |
| 1      | Neha     | Executive   | P101       |
| 1      | Neha     | Executive   | P102       |
| 2      | Asha     | Engineering | P101       |

## **10. Real-World Understanding – When to Use Normalization?**

| Scenario                     | Recommended Approach             |
| ---------------------------- | -------------------------------- |
| Transactional systems (OLTP) | Normalization                    |
| Banking applications         | Normalization                    |
| Employee management          | Normalization                    |
| Order management             | Normalization                    |
| Data warehouses              | Often denormalized               |
| Reporting and analytics      | Denormalization where beneficial |

**Senior Developer / Solution Architect Perspective:**

- Use normalization to maintain data integrity and consistency.
- Avoid unnecessary duplicate data.
- Use appropriate indexes to improve JOIN performance.
- Consider denormalization only after identifying actual performance bottlenecks.
- Measure performance using execution plans and query statistics before changing the schema.

**Interview-ready conclusion:**

"Normalization helps us design a database with minimal redundancy and better data integrity. In OLTP applications, I generally prefer a normalized schema, typically up to 3NF. However, for reporting and read-heavy workloads, I evaluate denormalization based on actual query performance and business requirements."
