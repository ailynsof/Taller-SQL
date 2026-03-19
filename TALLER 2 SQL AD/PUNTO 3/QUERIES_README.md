# PUNTO 3 — Academic Season Analysis
## Queries Documentation
**Database:** Academic (AD) — Oracle Live SQL  
**Workshop:** SQL Academic Database Analysis  
**Date:** 2026

---

## TABLE OF CONTENTS
1. [Explore Schema](#1-explore-schema)
2. [Consolidated View — Full Outer Join](#2-consolidated-view)
3. [NULL Analysis — Missing Data](#3-null-analysis)
4. [Duplicate Detection](#4-duplicate-detection)
5. [Faculties Needing Strengthening](#5-faculties-analysis)
6. [Additional Analysis Queries](#6-additional-queries)

---

## 1. EXPLORE SCHEMA

```sql
-- List all tables in AD schema
SELECT table_name 
FROM all_tables 
WHERE owner = 'AD' 
ORDER BY table_name;

-- Columns of AD_ACADEMIC_SESSION
SELECT column_name, data_type
FROM all_tab_columns
WHERE owner = 'AD' AND table_name = 'AD_ACADEMIC_SESSION';

-- Columns of AD_COURSE_DETAILS
SELECT column_name, data_type
FROM all_tab_columns
WHERE owner = 'AD' AND table_name = 'AD_COURSE_DETAILS';

-- Columns of AD_DEPARTMENTS
SELECT column_name, data_type
FROM all_tab_columns
WHERE owner = 'AD' AND table_name = 'AD_DEPARTMENTS';

-- Columns of AD_FACULTY_DETAILS
SELECT column_name, data_type
FROM all_tab_columns
WHERE owner = 'AD' AND table_name = 'AD_FACULTY_DETAILS';

-- Columns of AD_FACULTY_COURSE_DETAILS
SELECT column_name, data_type
FROM all_tab_columns
WHERE owner = 'AD' AND table_name = 'AD_FACULTY_COURSE_DETAILS';

-- Preview each table
SELECT * FROM AD.AD_ACADEMIC_SESSION;
SELECT * FROM AD.AD_COURSE_DETAILS;
SELECT * FROM AD.AD_DEPARTMENTS;
SELECT * FROM AD.AD_FACULTY_DETAILS;
SELECT * FROM AD.AD_FACULTY_COURSE_DETAILS;
SELECT * FROM AD.AD_JOBS;
```

---

## 2. CONSOLIDATED VIEW

> Note: CREATE VIEW was not used due to schema-level privilege restrictions on the AD schema
> in Oracle Live SQL. A WITH clause (CTE) was used instead, which produces equivalent results.

### 2A. Full Outer Join — All Data in One Table

```sql
SELECT 
    s.SESSION_ID,
    s.SESSION_NAME,
    d.DEPARTMENT_ID,
    d.DEPARTMENT_NAME,
    d.HOD,
    c.COURSE_ID,
    c.COURSE_NAME,
    f.FACULTY_ID,
    f.FIRST_NAME || ' ' || f.LAST_NAME  AS FACULTY_NAME,
    f.JOB_ID,
    f.HIRE_DATE,
    f.SALARY
FROM AD.AD_ACADEMIC_SESSION s
FULL OUTER JOIN AD.AD_COURSE_DETAILS         c  ON s.SESSION_ID    = c.SESSION_ID
FULL OUTER JOIN AD.AD_DEPARTMENTS            d  ON c.DEPARTMENT_ID = d.DEPARTMENT_ID
FULL OUTER JOIN AD.AD_FACULTY_COURSE_DETAILS fc ON c.COURSE_ID     = fc.COURSE_ID
FULL OUTER JOIN AD.AD_FACULTY_DETAILS        f  ON fc.FACULTY_ID   = f.FACULTY_ID
ORDER BY s.SESSION_ID, d.DEPARTMENT_ID, f.FACULTY_ID;
```
**Result:** 15 records — Spring Session (5), Fall Session (5), Summer Session (5)

---

## 3. NULL ANALYSIS

### 3A. Show all records with any NULL field

```sql
WITH CONSOLIDADO AS (
    SELECT 
        s.SESSION_ID,
        s.SESSION_NAME,
        d.DEPARTMENT_ID,
        d.DEPARTMENT_NAME,
        d.HOD,
        c.COURSE_ID,
        c.COURSE_NAME,
        f.FACULTY_ID,
        f.FIRST_NAME || ' ' || f.LAST_NAME AS FACULTY_NAME,
        f.JOB_ID,
        f.HIRE_DATE,
        f.SALARY
    FROM AD.AD_ACADEMIC_SESSION s
    FULL OUTER JOIN AD.AD_COURSE_DETAILS         c  ON s.SESSION_ID    = c.SESSION_ID
    FULL OUTER JOIN AD.AD_DEPARTMENTS            d  ON c.DEPARTMENT_ID = d.DEPARTMENT_ID
    FULL OUTER JOIN AD.AD_FACULTY_COURSE_DETAILS fc ON c.COURSE_ID     = fc.COURSE_ID
    FULL OUTER JOIN AD.AD_FACULTY_DETAILS        f  ON fc.FACULTY_ID   = f.FACULTY_ID
)
SELECT * FROM CONSOLIDADO
WHERE SESSION_NAME    IS NULL
   OR DEPARTMENT_NAME IS NULL
   OR COURSE_NAME     IS NULL
   OR FACULTY_NAME    IS NULL;
```
**Result:** No items to display — 0 NULL values found

### 3B. Count NULLs per column

```sql
WITH CONSOLIDADO AS (
    SELECT 
        s.SESSION_NAME,
        d.DEPARTMENT_NAME,
        c.COURSE_NAME,
        f.FIRST_NAME || ' ' || f.LAST_NAME AS FACULTY_NAME
    FROM AD.AD_ACADEMIC_SESSION s
    FULL OUTER JOIN AD.AD_COURSE_DETAILS         c  ON s.SESSION_ID    = c.SESSION_ID
    FULL OUTER JOIN AD.AD_DEPARTMENTS            d  ON c.DEPARTMENT_ID = d.DEPARTMENT_ID
    FULL OUTER JOIN AD.AD_FACULTY_COURSE_DETAILS fc ON c.COURSE_ID     = fc.COURSE_ID
    FULL OUTER JOIN AD.AD_FACULTY_DETAILS        f  ON fc.FACULTY_ID   = f.FACULTY_ID
)
SELECT 
    COUNT(*)                                                   AS TOTAL_RECORDS,
    SUM(CASE WHEN SESSION_NAME    IS NULL THEN 1 ELSE 0 END)   AS NULLS_SESSION,
    SUM(CASE WHEN DEPARTMENT_NAME IS NULL THEN 1 ELSE 0 END)   AS NULLS_DEPARTMENT,
    SUM(CASE WHEN COURSE_NAME     IS NULL THEN 1 ELSE 0 END)   AS NULLS_COURSE,
    SUM(CASE WHEN FACULTY_NAME    IS NULL THEN 1 ELSE 0 END)   AS NULLS_FACULTY
FROM CONSOLIDADO;
```
**Result:** TOTAL=15, all NULL columns = 0

---

## 4. DUPLICATE DETECTION

### 4A. Duplicate courses

```sql
SELECT COURSE_NAME, COUNT(*) AS TIMES
FROM AD.AD_COURSE_DETAILS
GROUP BY COURSE_NAME
HAVING COUNT(*) > 1
ORDER BY TIMES DESC;
```
**Result:** No items to display — 0 duplicate courses

### 4B. Duplicate departments

```sql
SELECT DEPARTMENT_NAME, COUNT(*) AS TIMES
FROM AD.AD_DEPARTMENTS
GROUP BY DEPARTMENT_NAME
HAVING COUNT(*) > 1;
```
**Result:** No items to display — 0 duplicate departments

### 4C. Duplicate sessions

```sql
SELECT SESSION_NAME, COUNT(*) AS TIMES
FROM AD.AD_ACADEMIC_SESSION
GROUP BY SESSION_NAME
HAVING COUNT(*) > 1;
```
**Result:** No items to display — 0 duplicate sessions

---

## 5. FACULTIES ANALYSIS

### 5A. Courses per department (identify weakest)

```sql
SELECT 
    d.DEPARTMENT_NAME,
    d.HOD,
    COUNT(c.COURSE_ID) AS TOTAL_COURSES
FROM AD.AD_DEPARTMENTS d
LEFT JOIN AD.AD_COURSE_DETAILS c ON d.DEPARTMENT_ID = c.DEPARTMENT_ID
GROUP BY d.DEPARTMENT_NAME, d.HOD
ORDER BY TOTAL_COURSES ASC;
```
**Result:** Literature=3, Others=4 each → Literature needs strengthening

### 5B. Teaching load per faculty member

```sql
SELECT 
    f.FIRST_NAME || ' ' || f.LAST_NAME AS FACULTY_NAME,
    f.JOB_ID,
    COUNT(fc.COURSE_ID) AS COURSES_ASSIGNED
FROM AD.AD_FACULTY_DETAILS f
LEFT JOIN AD.AD_FACULTY_COURSE_DETAILS fc ON f.FACULTY_ID = fc.FACULTY_ID
GROUP BY f.FIRST_NAME, f.LAST_NAME, f.JOB_ID
ORDER BY COURSES_ASSIGNED ASC;
```
**Result:** 5 faculty with 1 course, 5 faculty with 2 courses

### 5C. Faculty with no courses assigned

```sql
SELECT 
    f.FACULTY_ID,
    f.FIRST_NAME || ' ' || f.LAST_NAME AS FACULTY_NAME,
    f.JOB_ID
FROM AD.AD_FACULTY_DETAILS f
LEFT JOIN AD.AD_FACULTY_COURSE_DETAILS fc ON f.FACULTY_ID = fc.FACULTY_ID
WHERE fc.COURSE_ID IS NULL;
```
**Result:** No items to display — all faculty have at least 1 course

---

## 6. ADDITIONAL QUERIES

### 6A. Average salary by job role

```sql
SELECT 
    f.JOB_ID,
    COUNT(DISTINCT f.FACULTY_ID)                    AS TOTAL_FACULTY,
    ROUND(AVG(f.SALARY), 2)                         AS AVG_SALARY,
    MIN(f.SALARY)                                   AS MIN_SALARY,
    MAX(f.SALARY)                                   AS MAX_SALARY
FROM AD.AD_FACULTY_DETAILS f
GROUP BY f.JOB_ID
ORDER BY AVG_SALARY DESC;
```

### 6B. Faculty hiring timeline

```sql
SELECT 
    f.FIRST_NAME || ' ' || f.LAST_NAME AS FACULTY_NAME,
    f.JOB_ID,
    f.HIRE_DATE,
    f.SALARY,
    ROUND((SYSDATE - f.HIRE_DATE) / 365, 1)        AS YEARS_AT_UNIVERSITY
FROM AD.AD_FACULTY_DETAILS f
ORDER BY f.HIRE_DATE ASC;
```

### 6C. Department activity per session

```sql
SELECT 
    s.SESSION_NAME,
    d.DEPARTMENT_NAME,
    COUNT(c.COURSE_ID)  AS COURSES_IN_SESSION
FROM AD.AD_ACADEMIC_SESSION s
JOIN AD.AD_COURSE_DETAILS   c ON s.SESSION_ID    = c.SESSION_ID
JOIN AD.AD_DEPARTMENTS      d ON c.DEPARTMENT_ID = d.DEPARTMENT_ID
GROUP BY s.SESSION_NAME, d.DEPARTMENT_NAME
ORDER BY s.SESSION_NAME, COURSES_IN_SESSION DESC;
```

### 6D. Courses per role (load balance check)

```sql
SELECT 
    f.JOB_ID,
    COUNT(DISTINCT f.FACULTY_ID)                                    AS TOTAL_FACULTY,
    COUNT(fc.COURSE_ID)                                             AS TOTAL_COURSES,
    ROUND(COUNT(fc.COURSE_ID) / 
          NULLIF(COUNT(DISTINCT f.FACULTY_ID), 0), 2)               AS COURSES_PER_FACULTY
FROM AD.AD_FACULTY_DETAILS f
LEFT JOIN AD.AD_FACULTY_COURSE_DETAILS fc ON f.FACULTY_ID = fc.FACULTY_ID
GROUP BY f.JOB_ID
ORDER BY COURSES_PER_FACULTY ASC;
```

---

## VIEWS SUMMARY

| View / CTE Name      | Description                              | Records |
|----------------------|------------------------------------------|---------|
| CONSOLIDADO (CTE)    | Full Outer Join — all tables merged      | 15      |
| NULL check           | Records with missing values              | 0       |
| Duplicate courses    | Courses with duplicate names             | 0       |
| Duplicate depts      | Departments with duplicate names         | 0       |
| Duplicate sessions   | Sessions with duplicate names            | 0       |
| Dept course count    | Total courses per department             | 4 rows  |
| Faculty load         | Courses assigned per faculty member      | 10 rows |

---

## KEY FINDINGS

| # | Finding | Impact |
|---|---------|--------|
| 1 | 0 NULLs in consolidated table | ✅ Excellent data quality |
| 2 | 0 duplicates across all tables | ✅ Clean database |
| 3 | Literature has only 3 courses vs 4 avg | ⚠️ Needs expansion |
| 4 | 5 faculty carry only 1 course | ⚠️ Potential rebalancing needed |
| 5 | No new faculty hired since 2015 | 🔴 Critical — 10-year gap |
| 6 | FA_HOD salary 2.5x higher than FA_AF | ℹ️ Salary review recommended |
