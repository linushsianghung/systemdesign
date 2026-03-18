# Schema Design

### Naming Convention
Database naming conventions are a subject of many debates, but `strong`, `consistent` conventions are a hallmark of a professional and maintainable project. The short answer is: **Consistency is the most important rule**. Whatever you choose, apply it everywhere.
However, there is a widely adopted set of *best practices* that has become the de facto standard in modern application development. I'll outline that for you and explain the reasoning.

#### The Recommended Convention
| Element         | Convention             | Example                                |
|:----------------|:-----------------------|:---------------------------------------|
| Tables          | Plural, snake_case     | urls, user_profiles, clicks            |
| Columns         | Singular, snake_case   | long_url, created_at, user_agent       |
| Primary Key     | Always just id         | id                                     |
| Foreign Key     | singular_table_name_id | url_id, user_profile_id                |
| All Identifiers | lowercase              | urls, long_url (never URLs or longUrl) |
| SQL Keywords    | UPPERCASE              | SELECT, FROM, CREATE TABLE, WHERE      |

1. **Tables: Plural vs. Singular**
    - **Recommendation**: **Plural** (`urls`)
    - **Reasoning**: This is the most intuitive approach. A table holds a *collection* of records. You `SELECT` from `urls` to get one or more URLs. It reads like plain English: *Get all records from urls where the id is 5*. This convention is heavily favored by most modern web frameworks (like Ruby on Rails, Django, and Laravel), so following it makes integration with those tools seamless.
2. **Word Separator: snake_case vs. camelCase**
    - **Recommendation**: `snake_case` (`user_agent`)
    - **Reasoning**: This is arguably the most critical convention to follow.
        - **Case-Insensitivity**: Many SQL databases (like PostgreSQL) will fold unquoted identifiers to lowercase by default. If you name a column `userAgent` (`camelCase`), a query like `SELECT userAgent FROM ...` might fail because the database sees it as `select useragent from ....` To make it work reliably, you'd have to quote it everywhere: `SELECT "userAgent" FROM ....` This is tedious, error-prone, and universally considered a bad practice.
        - **Readability**: `snake_case` is the traditional and most readable format within the SQL ecosystem. It also contrasts nicely with `UPPERCASE` SQL keywords.
3. **Casing: lowercase vs. UPPERCASE**
    - **Recommendation**: `lowercase` for identifiers, `UPPERCASE` for keywords.
    - **Reasoning**:
        - Using `lowercase` for table and column names completely avoids the case-sensitivity issues mentioned above. It's the safest, most portable convention.
        - Using `UPPERCASE` for SQL keywords (`SELECT`, `FROM`, etc.) provides excellent visual separation between the logic of your query and the schema identifiers. It makes queries significantly easier to read at a glance.
   ```sql
    /* Good */
    SELECT long_url, short_code FROM urls WHERE id = 123;
   
    /* Bad (Hard to read) */
    select long_url, short_code from urls where id = 123;
   ```

### Table Join
- Inner Join: INNER JOIN returns only the **rows that match in both tables**.
- OUTER JOIN: Outer Join returns **non-matching rows, filling missing values with NULL**.


### Table Schema Design
- [From Idea to Production-Ready Database Design](https://www.youtube.com/watch?v=lWX5mk2adrg)
- [7 Database Design Mistakes to Avoid](https://www.youtube.com/watch?v=s6m8Aby2at8)
    - Mistake 1 - Business Field as Primary Key
    - Mistake 2 - Storing Redundant Data
    - Mistake 3 - Spaces or Quotes in Table Names
    - Mistake 4 - Poor or no Referential Integrity
    - Mistake 5 - Multiple Pieces of Information in a Single Field
    - Mistake 6 - Storing Optional Types of Data in Different Columns
    - Mistake 7 - Using the Wrong Data Types and Sizes



### Q & A
- **What is the difference between DDL and DML?**<br>
DDL defines the **structure of the database** (`schemas`, `tables`, `indexes`), while DML operates on the **data inside those structures** (`insert`, `update`, `delete`, `query`).
>- **DDL (Data Definition Language)** is used to create, alter, or remove database objects.
>- **DML (Data Manipulation Language)** is used to read and modify the data stored in those objects.


**Why do we separate DDL and DML?**<br>
Because schema changes and data changes have different lifecycles, risks, and transactional behaviors. Separating DDL and DML helps databases optimize execution, locking, and recovery.

### Conclusion
> DDL statements usually cause an implicit commit, so they can’t be rolled back in most databases, while DML statements are transactional and can be committed or rolled back.

