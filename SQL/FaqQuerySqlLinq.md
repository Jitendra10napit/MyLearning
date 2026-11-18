Here are the queries interviewers ask most often, each in SQL and LINQ (EF Core), all using one schema.

**Schema**

```
Employees(Id, Name, Email, Salary, DepartmentId, ManagerId, JoiningDate)
Departments(Id, Name)
```

**1. Second highest salary**

```sql
SELECT MAX(Salary) FROM Employees
WHERE Salary < (SELECT MAX(Salary) FROM Employees);

-- or
SELECT DISTINCT Salary FROM Employees
ORDER BY Salary DESC
OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY;
```

```csharp
var second = db.Employees.Select(e => e.Salary).Distinct()
               .OrderByDescending(s => s).Skip(1).FirstOrDefault();
```

**2. Nth highest salary**

```sql
SELECT DISTINCT Salary FROM Employees
ORDER BY Salary DESC
OFFSET (@N - 1) ROWS FETCH NEXT 1 ROWS ONLY;

-- with DENSE_RANK
SELECT Name, Salary FROM (
  SELECT Name, Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS rnk
  FROM Employees) t
WHERE rnk = @N;
```

```csharp
var nth = db.Employees.Select(e => e.Salary).Distinct()
            .OrderByDescending(s => s).Skip(n - 1).FirstOrDefault();
```

**3. Highest salary per department**

```sql
SELECT d.Name, MAX(e.Salary) AS MaxSalary
FROM Employees e JOIN Departments d ON e.DepartmentId = d.Id
GROUP BY d.Name;
```

```csharp
var result = db.Employees
    .GroupBy(e => e.Department.Name)
    .Select(g => new { Dept = g.Key, MaxSalary = g.Max(e => e.Salary) });
```

**4. Employee(s) with the highest salary in each department**

```sql
SELECT Name, Salary, DepartmentId FROM (
  SELECT *, RANK() OVER (PARTITION BY DepartmentId ORDER BY Salary DESC) AS rnk
  FROM Employees) t
WHERE rnk = 1;
```

```csharp
var result = db.Employees
    .Where(e => e.Salary == db.Employees
                  .Where(x => x.DepartmentId == e.DepartmentId)
                  .Max(x => x.Salary));
```

**5. Top 3 salaries per department**

```sql
SELECT * FROM (
  SELECT e.Name, e.Salary, d.Name AS Dept,
         ROW_NUMBER() OVER (PARTITION BY e.DepartmentId ORDER BY e.Salary DESC) AS rn
  FROM Employees e JOIN Departments d ON e.DepartmentId = d.Id) t
WHERE rn <= 3;
```

```csharp
var result = db.Departments.Select(d => new {
    d.Name,
    Top3 = d.Employees.OrderByDescending(e => e.Salary).Take(3)
});
```

**6. Find duplicate records (same email)**

```sql
SELECT Email, COUNT(*) AS Cnt
FROM Employees
GROUP BY Email
HAVING COUNT(*) > 1;
```

```csharp
var dupes = db.Employees
    .GroupBy(e => e.Email)
    .Where(g => g.Count() > 1)
    .Select(g => new { Email = g.Key, Count = g.Count() });
```

**7. Delete duplicates, keeping one**

```sql
WITH cte AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY Email ORDER BY Id) AS rn
  FROM Employees)
DELETE FROM cte WHERE rn > 1;
```

```csharp
var keepIds = db.Employees.GroupBy(e => e.Email).Select(g => g.Min(e => e.Id));
await db.Employees.Where(e => !keepIds.Contains(e.Id)).ExecuteDeleteAsync();
```

**8. Employees earning more than their manager (self join)**

```sql
SELECT e.Name AS Employee, m.Name AS Manager
FROM Employees e
JOIN Employees m ON e.ManagerId = m.Id
WHERE e.Salary > m.Salary;
```

```csharp
var result = from e in db.Employees
             join m in db.Employees on e.ManagerId equals m.Id
             where e.Salary > m.Salary
             select new { Employee = e.Name, Manager = m.Name };
```

**9. Employee with their manager name, including those without a manager**

```sql
SELECT e.Name, ISNULL(m.Name, 'No Manager') AS Manager
FROM Employees e
LEFT JOIN Employees m ON e.ManagerId = m.Id;
```

```csharp
var result = from e in db.Employees
             join m in db.Employees on e.ManagerId equals m.Id into mgr
             from m in mgr.DefaultIfEmpty()
             select new { e.Name, Manager = m != null ? m.Name : "No Manager" };
```

**10. Departments with no employees**

```sql
SELECT d.Name FROM Departments d
LEFT JOIN Employees e ON d.Id = e.DepartmentId
WHERE e.Id IS NULL;

-- or
SELECT Name FROM Departments d
WHERE NOT EXISTS (SELECT 1 FROM Employees e WHERE e.DepartmentId = d.Id);
```

```csharp
var result = db.Departments.Where(d => !d.Employees.Any()).Select(d => d.Name);
```

**11. Employee count per department (including 0)**

```sql
SELECT d.Name, COUNT(e.Id) AS EmpCount
FROM Departments d
LEFT JOIN Employees e ON d.Id = e.DepartmentId
GROUP BY d.Name;
```

```csharp
var result = db.Departments.Select(d => new { d.Name, Count = d.Employees.Count() });
```

`COUNT(e.Id)` counts only matched rows, while `COUNT(*)` would give 1 for empty departments. Interviewers check for this.

**12. Departments with more than 5 employees**

```sql
SELECT DepartmentId, COUNT(*) AS Cnt
FROM Employees
GROUP BY DepartmentId
HAVING COUNT(*) > 5;
```

```csharp
var result = db.Employees.GroupBy(e => e.DepartmentId)
               .Where(g => g.Count() > 5)
               .Select(g => new { g.Key, Count = g.Count() });
```

**13. Employees earning above their department average**

```sql
SELECT e.Name, e.Salary FROM Employees e
WHERE e.Salary > (SELECT AVG(Salary) FROM Employees x
                  WHERE x.DepartmentId = e.DepartmentId);
```

```csharp
var result = db.Employees.Where(e => e.Salary >
    db.Employees.Where(x => x.DepartmentId == e.DepartmentId).Average(x => x.Salary));
```

**14. Department summary (count, min, max, avg, total)**

```sql
SELECT d.Name, COUNT(*) Cnt, MIN(Salary) MinSal, MAX(Salary) MaxSal,
       AVG(Salary) AvgSal, SUM(Salary) Total
FROM Employees e JOIN Departments d ON e.DepartmentId = d.Id
GROUP BY d.Name;
```

```csharp
var result = db.Employees.GroupBy(e => e.Department.Name).Select(g => new {
    Dept = g.Key, Count = g.Count(), Min = g.Min(e => e.Salary),
    Max = g.Max(e => e.Salary), Avg = g.Average(e => e.Salary), Total = g.Sum(e => e.Salary)
});
```

**15. Running total (cumulative salary)**

```sql
SELECT Name, Salary,
       SUM(Salary) OVER (ORDER BY Id) AS RunningTotal
FROM Employees;
```

```csharp
// EF can't translate this; do it in memory
decimal total = 0;
var result = (await db.Employees.OrderBy(e => e.Id).ToListAsync())
             .Select(e => new { e.Name, e.Salary, RunningTotal = total += e.Salary });
```

**16. Pagination**

```sql
SELECT * FROM Employees
ORDER BY Id
OFFSET (@Page - 1) * @Size ROWS FETCH NEXT @Size ROWS ONLY;
```

```csharp
var page = db.Employees.OrderBy(e => e.Id).Skip((p - 1) * size).Take(size);
```

**17. Employees who joined in the last 6 months, or in a given year**

```sql
SELECT * FROM Employees WHERE JoiningDate >= DATEADD(MONTH, -6, GETDATE());
SELECT * FROM Employees WHERE YEAR(JoiningDate) = 2025;
```

```csharp
var recent = db.Employees.Where(e => e.JoiningDate >= DateTime.UtcNow.AddMonths(-6));
var y2025  = db.Employees.Where(e => e.JoiningDate.Year == 2025);
```

**18. Names starting with 'A' / containing a word**

```sql
SELECT * FROM Employees WHERE Name LIKE 'A%';
SELECT * FROM Employees WHERE Name LIKE '%kumar%';
```

```csharp
db.Employees.Where(e => e.Name.StartsWith("A"));
db.Employees.Where(e => e.Name.Contains("kumar"));
```

**19. Ranking: ROW_NUMBER vs RANK vs DENSE_RANK (asked very often)**

```sql
SELECT Name, Salary,
  ROW_NUMBER() OVER (ORDER BY Salary DESC) AS RowNum,
  RANK()       OVER (ORDER BY Salary DESC) AS Rnk,
  DENSE_RANK() OVER (ORDER BY Salary DESC) AS DenseRnk
FROM Employees;
```

| Salary | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|
| 90000 | 1 | 1 | 1 |
| 85000 | 2 | 2 | 2 |
| 85000 | 3 | 2 | 2 |
| 70000 | 4 | **4** | **3** |

- **ROW_NUMBER**: always unique.
- **RANK**: ties share a rank, then it skips numbers.
- **DENSE_RANK**: ties share a rank, with no gaps.

**20. Organization hierarchy (recursive CTE)**

```sql
WITH Org AS (
  SELECT Id, Name, ManagerId, 0 AS Level FROM Employees WHERE ManagerId IS NULL
  UNION ALL
  SELECT e.Id, e.Name, e.ManagerId, o.Level + 1
  FROM Employees e JOIN Org o ON e.ManagerId = o.Id)
SELECT * FROM Org ORDER BY Level;
```

LINQ has no recursion that EF can translate. Use `FromSql` with this CTE, or load the data and recurse in C#.

**21. Second highest salary per department**

```sql
SELECT * FROM (
  SELECT Name, Salary, DepartmentId,
         DENSE_RANK() OVER (PARTITION BY DepartmentId ORDER BY Salary DESC) AS rnk
  FROM Employees) t
WHERE rnk = 2;
```

```csharp
var result = db.Employees.GroupBy(e => e.DepartmentId).Select(g => new {
    Dept = g.Key,
    Second = g.Select(e => e.Salary).Distinct().OrderByDescending(s => s).Skip(1).FirstOrDefault()
});
```

**22. Swap or update values: give the IT department a 10% raise**

```sql
UPDATE e SET e.Salary = e.Salary * 1.10
FROM Employees e JOIN Departments d ON e.DepartmentId = d.Id
WHERE d.Name = 'IT';
```

```csharp
await db.Employees.Where(e => e.Department.Name == "IT")
        .ExecuteUpdateAsync(s => s.SetProperty(e => e.Salary, e => e.Salary * 1.10m));
```

**23. Odd or even rows**

```sql
SELECT * FROM Employees WHERE Id % 2 = 0;   -- even
```

```csharp
db.Employees.Where(e => e.Id % 2 == 0);
```

**24. UNION vs UNION ALL**

```sql
SELECT Name FROM Employees
UNION        -- removes duplicates (slower)
SELECT Name FROM Contractors;

SELECT Name FROM Employees
UNION ALL    -- keeps duplicates (faster)
SELECT Name FROM Contractors;
```

```csharp
db.Employees.Select(e => e.Name).Union(db.Contractors.Select(c => c.Name));    // UNION
db.Employees.Select(e => e.Name).Concat(db.Contractors.Select(c => c.Name));   // UNION ALL
```

**25. Theta join: salary grades by range**

```sql
SELECT e.Name, g.Grade FROM Employees e
JOIN SalaryGrades g ON e.Salary BETWEEN g.MinSal AND g.MaxSal;
```

```csharp
var result = from e in db.Employees
             from g in db.SalaryGrades
             where e.Salary >= g.MinSal && e.Salary <= g.MaxSal
             select new { e.Name, g.Grade };
```

**Short theory questions that come with these**

| Question | Short answer |
|---|---|
| WHERE vs HAVING | WHERE filters rows before grouping; HAVING filters groups after |
| DELETE vs TRUNCATE vs DROP | DELETE removes rows (logged, can use WHERE); TRUNCATE removes all rows fast (resets identity); DROP removes the table |
| INNER vs LEFT JOIN | INNER returns matches only; LEFT returns all left rows, plus matches or NULL |
| Clustered vs non-clustered index | Clustered = the physical row order (one per table); non-clustered = a separate lookup structure (many) |
| IN vs EXISTS | EXISTS stops at the first match, so it's often faster on big subqueries |
| CTE vs temp table | A CTE exists for one query only; a temp table is stored in tempdb and reusable in the session |
| `First()` vs `FirstOrDefault()` vs `Single()` | First throws if empty; FirstOrDefault returns null/default; Single throws if there's not exactly one |
| IQueryable vs IEnumerable | IQueryable builds SQL and runs in the DB; IEnumerable runs in memory |
| Deferred execution | LINQ runs only when enumerated (`ToList`, `foreach`, `Count`) |
| `Select` vs `SelectMany` | Select gives one output per item; SelectMany flattens nested collections |
| N+1 problem | A loop that triggers a query per item. Fix it with `Include` or a projection |
| `AsNoTracking()` | Skips change tracking, so read-only queries are faster |

**The ones asked almost every time:** #1, #2, #5, #6, #7, #8, #10, #19. Practice those until you can write them without looking.
