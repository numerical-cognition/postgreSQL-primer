# PostgreSQL Primer

This is work-in-progress, suggestions or tips are welcome.

[PostgreSQL](https://www.postgresql.org/) is an open source object-relational database system. It is a multi-user, multi-threaded database management system.  


# Installation and configuration  

## Install PostgreSQL on Debian/Ubuntu  


Add APT repository

```bash
# Create the file repository configuration:
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

Install PostgreSQL  

```bash
# Update the package lists:
$ sudo apt-get update

# Install the latest version of PostgreSQL.
# If you want a specific version, use 'postgresql-12' or similar instead of 'postgresql':
$ sudo apt-get -y install postgresql
```

## Install PostgreSQL on on macOS

```bash
$ brew install postgresql
```  

## Set password for postgres (PostgreSQL root) role

```bash
$ sudo -u postgres psql
\password
```

## Start, stop, and status of PostgreSQL database

Check the status of the database. Whether it's running or not.  

```bash
$ sudo service postgresql status
```

Start the database 

```bash
$ sudo service postgresql start
```

stop the database

```bash
$ sudo service postgresql stop
```


## Disable PostgreSQL database initialization on start-up

Each PostgreSQL cluster in Debian/Ubuntu has a start.conf file that controls what /etc/init.d/postgresql should do. Replace auto with manual in /etc/postgresql/13.x/main/start.conf.

## Disable auto boot  

```bash
$ update-rc.d -f postgresql remove
```

## Show clusters  

```bash
$ sudo pg_lsclusters
```

## Upgrade PostgreSQL

Install the newest PostgreSQL version. Check if you have more than one cluster running.

```bash
$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
12  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-12-main.log
13  main    5433 online postgres /var/lib/postgresql/13/main  /var/log/postgresql/postgresql-13-main.log
```

Stop the newer cluster

```bash
$ sudo pg_dropcluster 13 main --stop
```

Upgrade the older cluster to the newset version  

```bash
$ sudo pg_upgradecluster 12 main
```

Remove the older cluster

```bash
$ sudo pg_dropcluster 12 main
```

## User


A role is a user in a database world. Roles are separate from operating system users and global across a cluster. Users without password can connect to the cluster only locally (i.e. through socket).

## Start `psql`

```bash
$ sudo -u postgres psql
\password for sudo
```

After you enter `psql` for default user (which is called postgres), you can add more users/roles. 

## Create role  

On the command line

```bash
$ sudo -u postgres createuser [role_name] -d -P
```

In `psql`

```sql
$ CREATE ROLE [role_name] WITH LOGIN CREATEDB PASSWORD '[password]';
```

## List Roles

In `psql`

```bash
\du
```

or

```sql
SELECT * from pg_roles
```

## Change a user’s password  

```sql
ALTER ROLE [role_name] WITH PASSWORD '[new_password]';
```

## Allow user to create databases

```sql
ALTER USER <username> WITH CREATEDB;
```

****

# Database  

## List all databases

in psql

```bash
\l
```

## Create database


on the command line

```bash
$sudo -u postgres createdb [name] -O [role_name]
```
in psql

```sql
CREATE DATABASE [name] OWNER [role_name];
```

## Drop database

on the command line

```bash
sudo -u postgres dropdb [name] --if-exists
```

in psql

```sql
DROP DATABASE IF EXISTS [name];
```

## Export database as CSV

```sql
COPY (SELECT * FROM widgets) TO '/absolute/path/to/export.csv'
WITH FORMAT csv, HEADER true;
```

`COPY` requires to specify an absolute path and only writes to the file system local to the database. `\copy` is a wrapper around COPY that circumvents those restrictions.

```sql
\copy (SELECT * FROM widgets) TO export.csv
WITH FORMAT csv, HEADER true
```

You can specify encoding of exported data (e.g. latin1 for Excel instead of utf-8).

```sql
\copy (SELECT * FROM widgets) TO export.csv
WITH FORMAT csv, HEADER true, ENCODING 'latin1'
```

## Create compressed PostgreSQL database backup

```bash
pg_dump -U [role_name] [db_name] -Fc > backup.dump
```

## Create schema-only database backup

Schema-only means tables, indexes, triggers, etc, but not data inside; done -s option.

```bash
pg_dump -U [role_name] [db_name] -s > schema.sql
```

## Restore database from binary dump

```sql
PGPASSWORD=<password> pg_restore -Fc --no-acl --no-owner -U <user> -d <database> <filename.dump>
```

## Create compressed backups for all databases at once

```bash
pg_dump -Fc
```

## Convert binary database dump to SQL file

```bash
pg_restore binary_file.backup > sql_file.sql
```

## Copy database quickly

In order to copy a database, we create a new database and specify an existing one as the template.

```bash
createdb -T app_db app_db_backup
```

## Change database ownership

```sql
ALTER DATABASE acme OWNER TO zaiste;
```

# Table

## List tables

```bash
\dt
```

## Create table

```sql
CREATE TABLE widgets (
  id BIGINT PRIMARY KEY,
  name VARCHAR(20),
  price INT,
  created_at timestamp without time zone default now()
)
```

## Insert into table

```sql
INSERT INTO widgets VALUES(1,'widget1',100)
INSERT INTO widgets(name, price) VALUES ('widget2', 101)
```

## Upsert into table

PostgreSQL 9.5 or newer:

```sql
INSERT INTO widgets VALUES ... ON CONFLICT UPDATE
```

PostgreSQL 9.4 (and older) doesn't have built-in UPSERT (or MERGE) facility and it's difficult to do it right (concurrent use).

## Drop table

```sql
DROP TABLE IF EXISTS widgets
```

## Delete all rows from table

```sql
DELETE FROM widgets;
```

## Drop table and dependencies

```sql
DROP TABLE table_name CASCADE;
```

# Column

## Available column types

## Create Enum type

```sql
CREATE TYPE environment AS ENUM ('development', 'staging', 'production');
```

## Add column to table

```sql
ALTER TABLE [table_name] ADD COLUMN [column_name] [data_type];
```

## Remove column from table

```sql
ALTER TABLE [table_name] DROP COLUMN [column_name];
```

## Change column data type

```sql
ALTER TABLE [table_name] ALTER COLUMN [column_name] [data_type];
```

## Change column name

```sql
ALTER TABLE [table_name] RENAME COLUMN [column_name] TO [new_column_name];
```

## Set default value for existing column

```sql
ALTER TABLE [table_name] ALTER_COLUMN created_at SET DEFAULT now();
```

## Add `UNIQUE` constrain to existing column

```sql
ALTER TABLE [table_name] ADD CONSTRAINT [constraint_name] UNIQUE ([column_name]);
```
or shorter (PostgreSQL will automatically assign a constrain name)

```sql
ALTER TABLE [table_name] ADD UNIQUE ([column_name]);
```

# Date and time

## Dates

`clock_timestamp()` returns current value. `now()` always returns current value, unless in a transaction, in which case it returns the value from the beginning of the transaction.

## Convert datetimes between timezones

```sql
SELECT now() AT TIME ZONE 'GMT';
SELECT now() AT TIME ZONE 'PST';
```

## Get to know the day of the week for a given date

```sql
SELECT extract(DAY FROM now());
```

## Specify time interval
- 3 days ago: `SELECT now() - interval '3 days'`;
- 4 hours ago: `SELECT now() - interval '4 hours'`;
- 2 days and 7 hours ago: `SELECT now() - interval ‘2 days 7 hours'`;

# JSON

## Extract from JSON

```sql
SELECT '{"arr":[2,4,6,8]}'::json -> 'arr' -> 2              # returns 6
```
```sql
SELECT '{"arr":[2,4,6,8]}'::json #> ARRAY['arr','2']        # returns 6
```

## Create JSON array

```sql
CREATE TABLE test (
  j JSON,
  ja JSON[]
);
```

```sql
INSERT INTO test(ja) VALUES (
  array[
    '{"name":"alex", "age":20}'::json,
    '{"name":"peter", "age":24}'::json
  ]
);
```


# Indexes

An index helps retrieve rows from a table and its use is efficient only if the number of rows to be retrieved is relatively small. Adding an index to a column will allow you to query the data faster, but data inserts will be slower.

A partial index covers just a subset of a table’s data. It is an index with a WHERE clause. By reducing the index size (less storage, easier to maintain, faster to scan), its efficiency increases.

sequential scan: the database searches over all of the data before returning the results.

```sql
CREATE INDEX widgets_paid_index ON widgets(paid) WHERE paid IS TRUE;
```

# Types

- B-Tree is the default index (compatible with all data types) and it creates abalanced tree i.e. amount of data on both sides of the tree is roughly the same. Thenumber of levels that must be traversed to find rows is always similar. B-Tree indexesare well suited for equality and range queries.

- Hash indexes are only useful for equality comparisons and not transaction safe, theyneed to be manually rebuilt after crashes. There is no substantial advantage overusing B-Tree

- Generalized Inverted (GIN) is useful when an index maps many values to one row. Theyare good for indexing array values and for implementing full-text search.

- Generalized Search Tree (GiST) allows to build general balanced tree structures.They can be used not only for equality & and range comparisons operations. They areused to index the geometric data types and full-text search.


# Original document

[view original](https://zaiste.net/posts/postgresql-primer-for-busy-people/)
