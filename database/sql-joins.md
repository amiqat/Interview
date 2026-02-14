## SQL JOIN Types

**Difficulty:** Easy

**Topic:** SQL, Joins

### Problem Statement

Explain the different types of SQL JOINs and when to use each one. Provide examples.

### JOIN Types

**1. INNER JOIN**
- Returns only matching rows from both tables
- Most common type of join
- Default JOIN type in most databases

**2. LEFT JOIN (LEFT OUTER JOIN)**
- Returns all rows from left table
- Matching rows from right table (NULL if no match)
- Useful when you want all records from the main table

**3. RIGHT JOIN (RIGHT OUTER JOIN)**
- Returns all rows from right table
- Matching rows from left table (NULL if no match)
- Less common, can be rewritten as LEFT JOIN

**4. FULL OUTER JOIN**
- Returns all rows from both tables
- NULLs where there's no match
- Not supported in all databases (e.g., MySQL)

**5. CROSS JOIN**
- Cartesian product of both tables
- Every row from first table with every row from second
- Rarely used in practice

**6. SELF JOIN**
- Join a table to itself
- Useful for hierarchical data

### Examples

**Sample Tables:**

**employees table:**
| id | name | department_id | manager_id |
|----|------|---------------|------------|
| 1 | Alice | 1 | NULL |
| 2 | Bob | 1 | 1 |
| 3 | Carol | 2 | 1 |
| 4 | Dave | NULL | 2 |

**departments table:**
| id | name |
|----|----------|
| 1 | Engineering |
| 2 | Sales |
| 3 | Marketing |

**1. INNER JOIN Example:**

```sql
-- Get employees with their department names
SELECT e.name AS employee_name, d.name AS department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

**Result:**
| employee_name | department_name |
|---------------|-----------------|
| Alice | Engineering |
| Bob | Engineering |
| Carol | Sales |

**Explanation:** Dave is excluded because he has no department (NULL).

**2. LEFT JOIN Example:**

```sql
-- Get all employees with their department names (including those without departments)
SELECT e.name AS employee_name, d.name AS department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

**Result:**
| employee_name | department_name |
|---------------|-----------------|
| Alice | Engineering |
| Bob | Engineering |
| Carol | Sales |
| Dave | NULL |

**Explanation:** Dave appears with NULL department since LEFT JOIN includes all left table rows.

**3. RIGHT JOIN Example:**

```sql
-- Get all departments with their employees (including departments with no employees)
SELECT e.name AS employee_name, d.name AS department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

**Result:**
| employee_name | department_name |
|---------------|-----------------|
| Alice | Engineering |
| Bob | Engineering |
| Carol | Sales |
| NULL | Marketing |

**Explanation:** Marketing appears with NULL employee since RIGHT JOIN includes all right table rows.

**4. FULL OUTER JOIN Example:**

```sql
-- Get all employees and all departments, showing all combinations
SELECT e.name AS employee_name, d.name AS department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

**Result:**
| employee_name | department_name |
|---------------|-----------------|
| Alice | Engineering |
| Bob | Engineering |
| Carol | Sales |
| Dave | NULL |
| NULL | Marketing |

**5. CROSS JOIN Example:**

```sql
-- Get every possible pairing of employees with departments
SELECT e.name AS employee_name, d.name AS department_name
FROM employees e
CROSS JOIN departments d;
```

**Result:** 4 employees × 3 departments = 12 rows (every combination)

**6. SELF JOIN Example:**

```sql
-- Get employees with their manager names
SELECT e.name AS employee_name, m.name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Result:**
| employee_name | manager_name |
|---------------|--------------|
| Alice | NULL |
| Bob | Alice |
| Carol | Alice |
| Dave | Bob |

### When to Use Each JOIN

| JOIN Type | Use Case |
|-----------|----------|
| INNER JOIN | When you only want matching records |
| LEFT JOIN | When you want all records from the main table, even without matches |
| RIGHT JOIN | Rarely used; usually rewrite as LEFT JOIN |
| FULL OUTER JOIN | When you want all records from both tables |
| CROSS JOIN | When you need all combinations (rare) |
| SELF JOIN | When relating rows within the same table |

### Performance Considerations

1. **INNER JOIN** is typically fastest (fewer rows)
2. **Indexes** on join columns significantly improve performance
3. **LEFT JOIN** can be slower than INNER JOIN
4. **CROSS JOIN** can be very expensive (avoid on large tables)
5. Join order matters in complex queries

### Common Interview Questions

**Q: What's the difference between INNER JOIN and LEFT JOIN?**
A: INNER JOIN returns only matching rows. LEFT JOIN returns all rows from the left table, with NULLs for non-matching right table rows.

**Q: When would you use a self join?**
A: When you need to relate rows within the same table, such as employees-managers, organizational hierarchy, or finding duplicates.

**Q: Can you have multiple joins in one query?**
A: Yes, you can chain multiple joins:
```sql
SELECT *
FROM table1 t1
JOIN table2 t2 ON t1.id = t2.t1_id
JOIN table3 t3 ON t2.id = t3.t2_id;
```

### Follow-up Questions

1. How would you optimize a query with multiple joins?
2. What is a join without a condition (implicit cross join)?
3. How do indexes affect join performance?
4. What's the difference between WHERE and ON in joins?

### Tags

`sql` `database` `joins` `easy` `fundamental`
