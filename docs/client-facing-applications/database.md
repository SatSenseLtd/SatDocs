# Overview
The portal and api have access to SatSense InSAR and user data contained in a [PostgreSQL](https://www.postgresql.org/) database (or sometimes known as a "postgres" database).  We are currently using PostgreSQL version 11.

Since the InSAR and user data is geographically aware, we use the extension [PostGIS](https://postgis.net/).  PostGIS is a highly capable open source extension to PostgreSQL that allows efficient storage and queries of geographical datasets.

This page aims to be a reference for how to install, set up and use PostgreSQL and PostGIS.  It aims to include instructions specific to our use case, but also to be a generic reference as well.  All links to PostgreSQL documentation will be to version 11 documentation.

Contents:
1. [Installation of PostgreSQL and PostGIS](#1-installation-of-postgresql-and-postgis)
2. [User Creation](#2-user-creation) 
3. Tuning (TODO)
4. [Backing Up Procedure](#4-backing-up-procedure)
5. [SatSense Postgres Ingester](#5-satsense-postgres-ingester)
6. [Current Database Setup](#)

# 1. Installation of PostgreSQL and PostGIS

## Production on Centos 7

TODO

## PostgreSQL 12 + PostGIS on Ubuntu 20.04

### PostgreSQL

Install postgres via 

```commandline
sudo apt install postgresql postgresql-contrib
```

To check it's working correctly, you can log in to `psql` as postgres

```commandline
sudo su postgres
psql
```

### Change data directory

The following assumes a clean installation of postgres.

Open the config:

```commandline
vi /etc/postgresql/12/main/postgresql.conf 
```

and change the value `data_directory` to the location you'd like.  You also need to initialise this directory via:


```commandline
/usr/lib/postgresql/12/bin/initdb <data_directory>
```

The change requires a restart of the postgresql process:

```commandline
sudo systemctl restart postgresql
```

You can then log into `psql` and check it has been update by using the command `SHOW data_directory;`.

### Change authentication

By default, postgresql only allows peer authentication.  This means that the linux user name has to match the postgresql user name for you to log in.  To allow logging in by using a password, first set a password for the postgresql `postgres` user by running the following in `psql`:

```postgresql
ALTER ROLE postgres WITH PASSWORD 'new_password';
```

Then edit the file

```commandline
sudo vi /etc/postgresql/12/main/pg_hba.conf
```

and change all lines that are similar to:

```commandline
local   all             all                                     peer
```

to 

```commandline
local   all             all                                     md5
```

And restart the postgres service:

```commandline
sudo systemctl restart postgresql
```

You'll then be able to log in using a password, try using

```commandline
psql -U postgres
```

where `-U` means the postgres user name you are providing.

### PostGIS 

Add the ubuntugis ppa:

```commandline
sudo add-apt-repository ppa:ubuntugis/ppa
sudo apt-get update
```

And then postgis:

```commandline
sudo apt install postgis postgresql-12-postgis-3
```

# 2. User Creation and Permissions

In the following, replace text surrounded '<' and '>' with values you would like.  The following assumes that you are connected to postgresql via `psql` as a superuser (e.g. postgres). 

## Create New User

To create a new user:
```
\set usertograntaccessto '<portal_user>'
CREATE USER :usertograntaccessto WITH ENCRYPTED PASSWORD '<password>';
```
where `<password>` is sufficiently strong, it is recommended to use a random password generator. The line `\set usertograntaccessto '<portal_user>'` sets a variable in the `psql` session - the value is called with a preceding colon i.e. `:usertograntaccessto`.

## Grant Read Only Privileges

We assume that the variable `:usertograntaccessto` is set as per the "Create New User" section. To grant read only access to a database:
```
\c <db name>
\set dbname '<db name>'
GRANT CONNECT ON DATABASE :dbname TO :usertograntaccessto;
GRANT USAGE ON SCHEMA public TO :usertograntaccessto;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO :usertograntaccessto;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO :usertograntaccessto;
```
Note that it is recommended to connect to the database first - the last three lines don't specify which database to grant permissions to, rather it is the database you are connected to that defines this.

The line `ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO :usertograntaccessto;` will mean that if tables are added to the schema, then `:usertograntaccessto` will automatically be given the `SELECT` privilege.
 
```
Important
Default privileges will only be granted to tables added by the same user as who ran the line "ALTER DEFAULT PRIVILEGES...". That is, if "postgres" ran the line "ALTER DEFAULT PRIVILEGES...", then "postgres" must be the user that changes the schema.
```

## Grant Read Write Privileges

### SELECT, INSERT, UPDATE

We assume that the variable `:usertograntaccessto` is set as per the "Create New User" section. To grant `SELECT, INSERT, UPDATE` access to user database:
```
\c <db name>
\set dbname '<db name>'
GRANT CONNECT ON DATABASE :dbname TO :usertograntaccessto;
GRANT USAGE ON SCHEMA public TO :usertograntaccessto;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO :usertograntaccessto;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE ON TABLES TO :usertograntaccessto;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO :usertograntaccessto;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO :usertograntaccessto;
```
As above, it is recommended to connect to the database first (i.e. with the first line).  Also as above, lines starting with `ALTER DEFAULT PRIVILEGES` will mean that if tables are added to the schema, then `:usertograntaccessto` will automatically be given the correct privileges.
```
Important
Default privileges will only be granted to tables added by the same user as who ran the line "ALTER DEFAULT PRIVILEGES...". That is, if "postgres" ran the line "ALTER DEFAULT PRIVILEGES...", then "postgres" must be the user that changes the schema.
```

### SELECT, INSERT, UPDATE, DELETE
We note that some apps may also require the `DELETE` privilege.  For instance, the admin site has the ability to delete users, whereas the portal does not. To grant `SELECT, INSERT, UPDATE, DELETE` access to user database:
```
\c <db name>
\set dbname '<db name>'
GRANT CONNECT ON DATABASE :dbname TO :usertograntaccessto;
GRANT USAGE ON SCHEMA public TO :usertograntaccessto;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO :usertograntaccessto;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO :usertograntaccessto;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO :usertograntaccessto;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO :usertograntaccessto;
```

## Change Table Ownership

We finish by noting that Postgres Ingester requires greater access to tables than SELECT, INSERT, UPDATE and DELETE.  Because it runs manual vacuuming, the user needs to own the tables it vacuums.  You can change user ownership via:
```
ALTER TABLE <table_name> OWNER TO <new_owner>;
```
Note that it is easiest to run this command as a superuser (e.g. `postgres`).

## Query Current Permissions

Whilst connected to a you can use 

```psql
\dp
```

to query current permissions.  And

```psql
\ddp
```

to query the default permissions for all users.

# 3. Tuning

See https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server

Open the postgresql.conf file, on live this is 
```commandline
sudo vi /data/pgdata11/postgresql.conf
```

And change the following:
* You could increase `max_connections` (live is 200).  Much more than a few hundred, should consider connection pooling ([PgBouncer](https://www.pgbouncer.org/)).
* Increase `shared_buffers` (live is 8GB).
* Increase `work_mem` (live is 40MB).
* Increase `maintenance_work_mem` (live is 1GB).
* Increase `effective_cache_size`, advice ranges from 1/4 to 1/2 of RAM, live is 32GB.

# 4. Backing Up Procedure

## pg_dump

To back up our databases, we currently use the PostgreSQL utility [pg_dump](https://www.postgresql.org/docs/11/app-pgdump.html).  This produces a single file which can then be used to restore in another PostgreSQL instance.

### Example usage
```
pg_dump database_name -Fc -p 5440 -U postgres -v > name_of_backup.bak
``` 
In the above:
* `database_name` should be replaced with the name of the database that you would like to back up.
* `-Fc` denotes that pg_dump should use a custom format.  The format here is `.bak`, which is determined by the suffix of the out file name. For more options, see the [docs](https://www.postgresql.org/docs/11/app-pgdump.html).
* `-p 5440` denotes the port that the PostgreSQL is listening on.
* `-U postgres` is the user that `pg_dump` should connect to the database as.
* `-v` adds verbosity. A back up of the live InSAR database can take a long time (2 days).
* `name_of_backup.bak` should be replaced with the name of the back up file you'd like to create.

### Troubleshooting
* If you get an error like "server mismatch error" then it's possible that more than one version of `pg_dump` is installed.  Check that you are calling the correct version, it should be bundled with the postgres installation.  For example the current server's `pg_dump` is at `/usr/pgsql-11/bin/pg_dump`.

## Set up pg_dump cron job

It is possibly a good idea to set up a separate user for cron jobs:
```
sudo useradd cronuser
```

It might be a good idea to create a readonly user in postgreSQL as well:
```
CREATE USER cronuser WITH ENCRYPTED PASSWORD '<pg_password>';
GRANT CONNECT ON DATABASE <database_name> TO cronuser;
\c <database_name>
GRANT USAGE ON SCHEMA public TO cronuser;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO cronuser;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO cronuser;
GRANT USAGE ON SCHEMA topology TO cronuser;
GRANT SELECT ON ALL TABLES IN SCHEMA topology TO cronuser;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA topology TO cronuser;
```

It is not a good idea to put the cronuser `<pg_password>` in the cron file.  Instead, we create a `.pgpass` file.  As the linux user `cronuser`, create the file
```
touch ~/.pgpass
chmod 0600 ~/.pgpass
```
which should be populated with the structure `server:port:database:username:password`.  For example, inserting the line:
``` 
localhost:5440:*:cronuser:<pg_password>
```
will allow credentials to any database (assuming `cronuser` has the correct permissions).  

Now run
```
crontab -e
```
and insert the something like:
```
0 22 * * * /usr/pgsql-11/bin/pg_dump <database_name> -Fc -w -p 5440 -U cronuser > /path/to/backups/database_name_$(date +"\%Y\%m\%d").bak
```
which will back up the database everyday at 22:00.  Notice that we've escaped '%' and **make sure** you also enter a new line after the above line.  A lovely *feature* of crontab is that they don't run commands without a new line after them.

## Restoring a .bak file

For the following commands, you many need to use `-p <port> -U <username>`, the commands may also suffer a similar "server mismatch error" to `pg_dump`. First create a new database:
```
createdb -T template0 mynewdb
```
Here:
* `-T template0` means that nothing is created in the new database.  By default, `createdb` will use `template1` which may include some default schemas which you will not need.
* `mynewdb` should be replaced with name of the new database you'd like to create.  It should not exist.
Once the database has been created, you can restore the back up:
```
pg_restore -d mynewdb -v name_of_backup.bak
```
* `-d mynewdb` is the database name, as above.
* `-v` adds verbosity.
* `name_of_backup.bak` is the name of the back up file you'd like to restore.

We note that the back up will try to restore the user/role access defined on the original database.  The `pg_restore` function will show some errors if the users/roles don't exist.  You shouldn't need to worry, the data should have restored fine but you will need to add any user access that you need.

Sometimes the `pg_restore` command can fail with an error like:
```
pg_restore: [archiver] unsupported version (1.14) in file header
```
This may be due to using the version of pg_restore which is within your conda environment. Deactivate the conda environment and try rerunning the command.

After restoring the backup, it is recommended to run 
```
ANALYSE;
```
whilst connected to the database via `psql`. Here is some brief information about [ANALYSE](https://andreigridnev.com/blog/2016-04-01-analyze-reindex-vacuum-in-postgresql/):
```
PostgreSQL ANALYZE command collects statistics about specific table columns, entire table, or entire database. The PostgreSQL query planner then uses that data to generate efficient execution plans for queries.
```

# 5. SatSense PostgreSQL Ingester 
The [SatSense PostgreSQL Ingester](https://gitlab.com/SatSenseLtd/postgresingester) (SatPGI) handles all updating of the SatSense InSAR dataset.  Documentation on how to checkout a development/production instance of SatPGI is held in the project README. This chapter is intended to be supplementary to that README; the contents are:

1. Current Installation Details (TODO)
2. Setting Up A User (TODO)
3. Vacuuming.

## 3. Vacuuming
### Introduction
PostgreSQL is based on a "Multi Version Concurrency Control" (MVCC) architecture.  Here is quite a good post about MVCC: 
```
http://shiroyasha.io/multiversion-concurrency-control.html
```
The important take away is that when updating (or deleting) a row in the database, there can be two "versions" (called tuples) of that row.  A query will get a particular tuple depending on when the query is run.  

Some of the database transactions that SatPGI runs are very long running. Queries that are run in the duration of one of these transactions will not see any updated data; they will read the old tuples.  However, queries run after a transaction has finished will see the updated tuples and not be able to read the old tuples.

A priori, a tuple does not know if it is able to be read (known as live) or not (known as dead).  There needs to be a process which detects dead tuples and deletes them.  This process is known as "vacuuming".

By default, PostgreSQL automates a periodic vacuuming process (autovacuuming). However, if large updates or deletes are run then autovaccuming may not be enough and a database can grow massively as it will be storing multiple tuples for each row (effectively doubling/tripling the storage size of that row). 

To help mitigate this problem, SatPGI will manually run Vacuum statements after it has run large updates.  So a SatPGI user should not have to worry too much, however this section should help anyone who wants to debug/further investigate vacuuming.

For reference, the PostgreSQL documentation page for vacuuming can be found here:
```
https://www.postgresql.org/docs/11/sql-vacuum.html
```

### Analysing

Connect to the database using `psql` and then run the following to get statistics about each table:
```
SELECT relname AS TableName, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum FROM pg_stat_user_tables;
```
In the above query:
* n_live_tup denotes the number of live tuples in the table;
* n_dead_tup denotes the number of dead tuples in the table;
* last_vacuum is a time stamp of when vacuum was last manually run (or null for never);
* last_autovacuum is a time stamp of when vacuum was last automatically run (or null for never);

The above statistics are only an estimation and certain processes (such as vacuuming) can update these statistics (finding out exactly when these stats are updated are a TODO).

### SatPGI's vacuuming implementation

SatPGI applies a standard (as opposed to a "full") vacuum to points tables that have just had large updates run on them.  This type of vacuum does not apply an exclusive lock on the table (as opposed to full vacuum which does) but it can be less effective at releasing hard disk space back to the system.  Even if this is that case, the space can be still be used for new/updated rows for the table in question.

The vacuum process cannot be run inside a transaction, as a result we have defined a OutOfTransactionUnitOfWork object (see [unit_of_work.py](https://gitlab.com/SatSenseLtd/satsense-domain/-/blob/master/satsense_core/data/unit_of_work.py)) which will handle this process for us.

Another interesting feature of vacuuming is that it must be run by either:
* a database superuser (such as postgres);
* the owner of the database containing the table being vacuumed;
* the owner of the table being vacuumed.

In the interest of giving the least amount of privileges, we opt for the last option for the user that SatPGI uses to connect to the database with.  One can alter the owner of a table via the command:
```
ALTER TABLE <table_name> OWNER TO <new_owner>;
```
Note that it is easiest to run this command as a superuser (although it is possible to run it without the superuser privilege).
