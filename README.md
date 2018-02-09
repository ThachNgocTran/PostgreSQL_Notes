# PostgreSQL_Notes
Notes on my experience with PostgreSQL. Serve as a reminder just in case I forgot something.

#### Table of Contents

[1. UPSERT = INSERT + UPDATE](#tip1)  

<a name="tip1"></a>
## 1. UPSERT = INSERT + UPDATE

+ Insert literal values, update if already existing.

```sql
INSERT INTO driver
(id, date_created, name)
VALUES
(89, '2009-02-08 12:51', 'fahrer 89'),
(45, '2009-04-19 06:41', 'YYY'),
(9, '2009-05-29 17:38', 'XXX')
ON CONFLICT (id) DO UPDATE
SET date_created = EXCLUDED.date_created,
            name = EXCLUDED.name;
```

+ Insert literal values, do nothing if already existing.

```sql
INSERT INTO customers (name, email)
VALUES
('Microsoft', 'hotline@microsoft.com')
ON CONFLICT (name) 
DO NOTHING;
```

+ Insert values from `SELECT`.

```sql
INSERT INTO animals(nm, typ, tvi, tvf)
SELECT nm,typ,tvi,tvf
FROM json_po
ON CONFLICT ...
```
