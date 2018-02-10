# PostgreSQL_Notes
Notes on my experience with PostgreSQL. Serve as a reminder just in case I forgot something.

#### Table of Contents

[1. UPSERT = INSERT + UPDATE](#tip1)  
[2. UPDATE](#tip2)  
[3. TRUNCATE](#tip3)  
[4. SELECT TOP 10](#tip4)  
[5. DATE QUERY](#tip5)  
[6. DATABASE CONFIGURATION](#tip6)  
[7. IMPORT DATA](#tip7)  
[8. EXECUTE A SQL SCRIPT](#tip8)  
[9. UNION](#tip9)  
[10. PARTITIONING](#tip10)  
[11. TRIGGER](#tip11)  
[12. PYTHON CONNECTION TO POSTGRESQL](#tip12)  
[13. SCHEMA VS. DATABASE](#tip13)  
[14. SYSTEM COMMANDS](#tip14)  

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

<a name="tip2"></a>
## 2. UPDATE

```sql
UPDATE master_table
SET user_id = st.user_id
FROM tinytransactions st
WHERE 	st.updated >= 1518219378 AND 
		st.id IN (SELECT mt.id FROM master_table mt) AND 
        st.id = master_table.id;
```

<a name="tip3"></a>
## 3. TRUNCATE

```sql
TRUNCATE TABLE driver CASCADE;
```

<a name="tip4"></a>
## 4. SELECT TOP 10

```sql
select *
from scores
order by score desc
limit 10
```

<a name="tip5"></a>
## 5. DATE QUERY

+ Extract part of date.

```sql
SELECT date_part('year', b.date_created) AS year_num, ROUND(AVG(b.rating), 3), SUM(b.tour_value)
FROM booking b
```

+ Get Epoch for yesterday (in millisecond)

```sql
SELECT (extract(epoch from TIMESTAMP 'yesterday')*1000)
SELECT (extract(epoch from TIMESTAMP 'yesterday')*1000 + 86399999)
```

<a name="tip6"></a>
## 6. DATABASE CONFIGURATION

+ Allow all connections.

```bash
sudo nano /etc/postgresql/9.1/main/pg_hba.conf
```

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# IPv4 local connections:
host    all             all             0.0.0.0/0            md5
# IPv6 local connections:
host    all             all             ::0/0                 md5
```

+ Change IP binding.

```bash
sudo nano /etc/postgresql/10/main/postgresql.conf
```

```ini
listen_addresses = '*'
```

```bash
sudo nano /etc/postgresql/10/main/pg_hba.conf
```

```bash
host    all         all         0.0.0.0/0        md5
```

+ Restart the server to force re-read configurations.

```bash
sudo service postgresql restart
```

```sql
SELECT pg_reload_conf();
```



<a name="tip7"></a>
## 7. IMPORT DATA

+ Import CSV.

```bash
export PGPASSWORD=test
psql -h localhost -d mydb -U test -c "\copy page (id, page_title, last_modification)  from 'tablePage.csv' with delimiter as ',' csv header  encoding 'UTF-8'"
```

+ Via pgAdmin.

Right click on the table, click "Import/Export".

Note: When creating the table for uploading the CSV into it, be sure to **not** include any constraints (like foreign key, primary key) and index ==> **greatly speed up** the uploading of data!!! Later, the constraints and index can be created manually after the data is fully loaded into the database.

```sql
CREATE INDEX transactions_updated_idx
    ON public.transactions USING btree
    (updated)
    TABLESPACE pg_default;
```

<a name="tip8"></a>
## 8. EXECUTE A SQL SCRIPT

```bash
psql -h localhost -d mydb -U test -f "script.sql"
```

<a name="tip9"></a>
## 9. UNION

Without “ALL", duplicate values are removed.

```sql
SELECT COUNT(*) FROM category
UNION ALL
SELECT COUNT(*) FROM page
UNION ALL
SELECT COUNT(*) FROM page_category
UNION ALL
SELECT COUNT(*) FROM page_link;
```

<a name="tip10"></a>
## 10. PARTITIONING

In PostgreSQL 10, an improved version of partitioning called “declarative partitioning".

Problem when data is extremely large, e.g. 50 millions of rows. ⇒ the index grows large, too, but it can not be stored entirely in memory ⇒ part in memory, part on disk ⇒ So, whenever it is seeked or updated (insert new rows), the DB has to read some pages of the index on disk ⇒ slow!

Solution: Table partitioning ⇒ the resulting sub-tables’ indexes (if done correctly) will be kept in memory during insert. ⇒ Improved inserts and queries from being able to fit all indexes in memory.

The exact point at which partitioning should be considered instead of a large table depends on the workload and machine resources.

**Partitioning via table inheritance: (old way)**

1). Create the “parent” table, from which all of the partitions will inherit.
2). Create several “child” tables (each representing a partition of the data) that each inherit from the parent.
3). Add constraints to the partition tables to define the row values in each partition.
4). Create indexes on any parent and child tables individually. (Indexes do not propagate from the parent tables to child tables).
5). Write a suitable trigger function to the master table so that inserts into the parent table redirect into the appropriate partition table.
6). Create a trigger that calls the trigger function.
7). Remember to redefine the trigger function when the set of child tables changes.

Note: Child tables are permitted to have extra columns not present in the parent table.

Downsides:

+ INSERT and COPY commands do not automatically propagate data to other child tables in the inheritance hierarchy, but instead rely on triggers, resulting in slower inserts.
+ Substantial manual work is required to create and maintain child sub-tables. ⇒ Indexes, constraints, and many table maintenance commands need to be applied to child tables explicitly. This greatly complicates management tasks.

**Partitioning via “declarative partitioning": (new way)**

Currently, declarative partitioning supports RANGE and LIST partitions:

+ RANGE—partition the table into “ranges” defined by a key column or set of columns, with no overlap between the ranges of values assigned to different partitions. For example, device_id.
+ LIST—partition the table by explicitly listing which key values appear in each partition, and are limited to a single column. For example, device_type.

```sql
CREATE TABLE measurement (
 city_id int not null,
 logdate date not null,
 peaktemp int,
 unitsales int
) PARTITION BY RANGE (logdate);

CREATE TABLE measurement_y2006m02 PARTITION OF measurement
 FOR VALUES FROM ('2006–02–01') TO ('2006–03–01')

CREATE TABLE measurement_y2006m03 PARTITION OF measurement
 FOR VALUES FROM ('2006–03–01') TO ('2006–04–01')

…

CREATE INDEX ON measurement_y2006m02 (logdate);
CREATE INDEX ON measurement_y2006m03 (logdate);
```

Advantages:

Commands such as TRUNCATE and COPY now **propagate to child tables via execution on the partitioned table**. Additionally, users can insert data into underlying child tables via the partitioned table, since tuples (i.e., rows) now **automatically route to the right partition on INSERT and no longer depend on triggers**, as is the case with table inheritance.

Disadvantages: (FOR NOW, MAY CHANGE IN VERSION 11)

1). For example, child tables for the data need to exist before the data is inserted. 
2). Cannot create indexes on all partitions automatically. Indexes still need to be manually created on each partition.
3). Updates that would move a row from one partition to another will fail.
4). Row triggers must be defined on individual partitions.
5). Multi-dimensional partitioning is cumbersome (...)
6). Cannot create primary keys on partitions: meaning that foreign keys referencing partitioned tables are not supported, nor are foreign key references from a partitioned table to another table.
7). No support for enforcing uniqueness (or an exclusion constraint) across an entire partitioned table. Unique or exclusion constraints can only be created on individual partitions.

See [1] for original posting.

<a name="tip11"></a>
## 11. TRIGGER

Trigger uses Stored Procedure function to be called when an event happens, with certain conditions satisfied. A trigger function is **similar to an ordinary function, except that it does not take any arguments and has return value type trigger**. Automatic available variables: OLD, NEW, TG_WHEN, TG_TABLE_NAME, etc.

```sql


-- Whenever employee’s last name changes, we will log it into a separate table named employee_audits through a trigger.
CREATE TABLE employee_audits (
   id int4 serial primary key,
   employee_id int4 NOT NULL,
   last_name varchar(40) NOT NULL,
   changed_on timestamp(6) NOT NULL
)

CREATE OR REPLACE FUNCTION log_last_name_changes()
  RETURNS trigger AS
$BODY$
BEGIN
 IF NEW.last_name <> OLD.last_name THEN
   INSERT INTO employee_audits(employee_id,last_name,changed_on)
   VALUES(OLD.id,OLD.last_name,now());
 END IF;
 
 RETURN NEW;
END;
$BODY$

CREATE TRIGGER last_name_changes
  BEFORE UPDATE
  ON employees
  FOR EACH ROW
  EXECUTE PROCEDURE log_last_name_changes();
```

See [2] for original posting.

<a name="tip12"></a>
## 12. PYTHON CONNECTION TO POSTGRESQL

```bash
pip install psycopg2
```

```python
import psycopg2
```

<a name="tip13"></a>
## 13. SCHEMA VS. DATABASE

A schema is a higher level of organization, a container or namespace within a database. A schema contains a set of tables, views, stored procedures and triggers and so on.

We cannot query the two databases simultaneously from the psql command line. Imagine now that we take the two databases and put them inside a single SuperFoods database. We create a container, a schema, for each of the two original databases. Each schema has a unique name, "sausalito" and "petaluma", and this causes each of the tables in the schema to have a qualified name that includes the schema name.

```sql
sausalito.employees
sausalito.sales

petaluma.employees
petaluma.sales

select 'sausalito', sum(sales) from sausalito.sales where year=2010
UNION ALL
select 'petaluma', sum(sales) from petaluma.sales where year=2010;
```

Now we understand more about schema we can refine our notion about them. They are a namespace or rather a namespace within a database. PostgreSQL only sees one database here: SuperFoods.

Databases are physically separated and access control is managed at the connection level. If one PostgreSQL server instance is to house projects or users that should be separate and for the most part unaware of each other, it is therefore recommendable to put them into separate databases. If the projects or users are interrelated and should be able to use each other's resources they should be put in the same database, but possibly into separate schemas. Schemas are a purely logical structure and who can access what is managed by the privilege system.

See [3], [4] for original posting.

<a name="tip14"></a>
## 14. SYSTEM COMMANDS

+ Get version

```sql
SELECT version();
```

# References

[1] https://blog.timescale.com/scaling-partitioning-data-postgresql-10-explained-cd48a712a9a1?gi=ea309940dfc4  
[2] http://www.postgresqltutorial.com/creating-first-trigger-postgresql/  
[3] http://www.postgresqlforbeginners.com/2010/12/schema.html  
[4] https://stackoverflow.com/questions/28951786/postgresql-multiple-database-vs-multiple-schemas  
