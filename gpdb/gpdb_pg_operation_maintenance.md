## GPDB/PG Operation and Maintenance



### psql query  and  scripting  tool

```
 #executes a single SQL comman, -c command is non-interactive
 psql -c "SELECT current_time"
 
 # execute multiple commands, -f script file
 psql –f examples.sql
 
 #used in interactive mode
 psql
 
 #two types of command
 >meta-commands, command for psql client
 >SQL, sent to the database server
 
 #two types of help 
 > \? provides help on psql meta-commandsf
 > \h provides help on specifc SQL commands
```

useful  features  of  psql

```
#psql 
> \password     #ALTER USER postgres PASSWORD 'newpassword';
> Information functions   #https://www.postgresql.org/docs/10/static/functions-info.html
> Output formatting   #\x
> \timing   #Execution timing by using the 
> Input/Output and editing commands
> Automatic startup fles: .psqlrc
> Substitutable parameters ("variables")
> Access to the OS command line
```





### Exploring  the  Database

#### What version is the server

```
#psql 
select version();
```

#### What is the server uptime

```
#psql
SELECT date_trunc('second', (current_timestamp - pg_postmaster_start_time())) as uptime;
SELECT pg_postmaster_start_time();
```

####Locate the database server message log 

```
> beneath the data directory
> may be redirected to syslog
# /var/log/postgresql   
# $PGDATA/pg_log

#show the contents of server control file
pg_controldata $PGDATA
```

#### List databases on this database server

```
psql -l
#psql
\l
select * from pg_database ;
```

#### How much disk space does a database use

```
SELECT pg_database_size(current_database());
SELECT sum(pg_database_size(datname)) from pg_database;
```

#### How much disk space does a table use

```
select pg_relation_size('tabname');
select pg_total_relation_size('accounts');   #including index ...
\dt+ tabname    #only for pg
```

#### Which are my biggest tables

```
#for pg
select table_name, pg_relation_size(table_name) as size
from information_schema.tables 
where table_schema NOT IN ('information_schema',  'pg_catalog' )
ORDER BY size DESC LIMIT 10;

#for gpdb
select table_name, pg_relation_size(table_name) as size
from information_schema.tables 
where table_schema NOT IN ('information_schema',  'pg_catalog', 'gp_toolkit' )
ORDER BY size DESC LIMIT 10;
```

#### How  many  rows  in  a  table

```
#scan fulltable
SELECT count(*) FROM table;
SELECT count(1) FROM table;

#Quick estimate of the number of rows in a table
SELECT (CASE WHEN reltuples > 0 THEN 
           pg_relation_size('mytable')/(8192*relpages/reltuples)          
        ELSE 0          
        END)::bigint AS estimated_row_count  
FROM pg_class  
WHERE oid = 'mytable'::regclass;
#result is not accurate, just for big table??
```



### Configuration

#### Reading the Fine Manual

```
> SQL Command Reference, Client, and Server tools reference 
  http://www.postgresql.org/docs/9.0/interactive/reference.html
> Confguration 
  http://www.postgresql.org/docs/9.0/interactive/runtime-config.html
> Functions 
  http://www.postgresql.org/docs/9.0/interactive/functions.html
```

#### Changing parameters in your programs

```
#change the value of a setting during your session
SET work_mem = '16MB';   
show work_mem;
#for pg
select name, setting, reset_val, source from pg_settings where source='session';
#for gpdb
select name, setting, source from pg_settings where source='session';

RESET work_mem;
select * from pg_settings where name='work_mem';
RESET ALL;

#change the value of a setting during your transaction
SET LOCAL work_mem = '16MB'; 
```

#### Which  parameters  are  at  non--default  settings

```
#psql
select name, source, setting from pg_settings 
where source != 'default' and source != 'override'
```

#### Updating the parameter fle

```
> postgresql.conf
> pg_hba.conf
> pg_ident.conf

pg_ctl -D $PGDATA reload
```

#### Setting parameters for particular groups of users

```
#You can set parameters for each of the following
> Databasef
> User (named "Roles" by PostgreSQL)
> Database / User combination

ALTER DATABASE saas SET work_mem = '16MB';
ALTER ROLE user SET work_mem = '16MB';
ALTER ROLE user IN DATABASE saas SET work_mem = '16MB';
```

#### Adding an external module

```
> additional functions
> additional datatypes
> additional operators
> additional indexes

> Contrib— PostgreSQL "core" includes many functions.
    http://www.postgresql.org/docs/9.0/static/ contrib.html
> pgFoundry
    http://pgFoundry.org/
> Separate projects
    http://www.postgis.org
    
# Oracle compatibility 
   https://postgres.cz/wiki/Oracle_functionality
```



### Server Control

#### Start database server

```
#for pg
pg_ctl -D $PGDATA start
```

#### Stop database server

```
#for pg
pg_ctl -D $PGDATA -m fast stop 
pg_ctl -D $PGDATA stop -m immediate

#before stop server
CHECKPOINT
#or
psql -c "CHECKPOINT"
```

#### Reloading  the  server  confguration  files

```
pg_ctl -D $PGDATA reload
#psql
select pg_reload_conf();

#query sighup parameters
SELECT name, setting, unit, (source = 'default') as is_default
from pg_settings 
where context = 'signhup' and ( name like '%delay%' or name like '%timeout' ) and setting != '0';

#or restart server
pg_ctl -D datadir restart -m fast
```

#### Preventing new connections

```
ALTER DATABASE foo_db CONNECTION LIMIT 0;
ALTER USER foo CONNECTION LIMIT 0;

#or change hba.conf
local    all         all         reject
host     all         all         0.0.0.0/0    reject
#If you still want superuser access
local    all         postgres|gpadmin         ident
local    all         all         reject
host     all         all         0.0.0.0/0    reject
```

#### Restricting users to just one session each

```
#psql
ALTER ROLE fred CONNECTION LIMIT 1;

#query conn limit
select rolname, rolconnlimit from pg_roles where rolname='name';

#query online connections
select count(*) from pg_stat_activity where usename='name';
```

#### Pushing users off the system

```
#for pg10
select pg_terminate_backend(pid) from pg_stat_activity where usename='name';
#for pg<=9 and gpdb
select pg_terminate_backend(procpid) from pg_stat_activity where usename='name';

# a safer query 
SELECT count(pg_terminate_backend(procpid|pid))
FROM pg_stat_activity
WHERE usename NOT IN(SELECT usename FROM pg_user WHERE usesuper);
# more useful fileter
> WHERE application_name = 'myappname'
> WHERE waiting
> WHERE current_query = '<IDLE> in transaction'
> WHERE current_query = '<IDLE>'
```

#### Using multiple schemas

```
CREATE SCHEMA finance;
CREATE SCHEMA sales;
CREATE TABLE finance.month_end_snapshot(...);
select current_schema;
ALTER ROLE fiona SET search_path = 'finance';
ALTER ROLE sally SET search_path = 'sales';

REVOKE ALL ON SCHEMA finance FROM public;
GRANT ALL ON SCHEMA finance TO fiona; 
REVOKE ALL ON SCHEMA sales FROM public;
GRANT ALL ON SCHEMA sales TO sally;

#An alternate technique would be to allow one user to create privileges on only one schema
REVOKE ALL ON SCHEMA finance FROM public;
GRANT USAGE ON SCHEMA finance TO public; #fiona??
GRANT CREATE ON SCHEMA finance TO fiona; 
REVOKE ALL ON SCHEMA sales FROM public;
GRANT USAGE ON SCHEMA sales TO sally;
GRANT CREATE ON SCHEMA sales TO sally;

#grants for object
GRANT SELECT ON month_end_snapshot TO public;

#set default privileges
ALTER DEFAULT PRIVILEGES FOR USER fiona IN SCHEMA finance GRANT SELECT ON TABLES TO PUBLIC;
```

#### Giving users their own private database

```
#psql
create user fred;
create database fred owner = fred;
BEGIN;
REVOKE connect ON DATABASE  fred FROM public;
GRANT connect ON DATABASE fred TO fred;
COMMIT;

```



### Data

#### Generating test data

```
#generate a sequence of rows
SELECT * FROM generate_series(1,5);

#generate a list of dates
SELECT date(generate_series(now(), now() + '1 week', '1 day'));

#Columns
> Random integer  
    (random()*(2*10^9))::integer
> Random bigint
    (random()*(9*10^18))::bigint
> Random numeric data
    (random()*100.)::numeric(4,2);
> Random length string, up to a maximum length
    repeat('1',(random()*40)::integer) 
> Random length substring
    substr('abcdefghijklmnopqrstuvwxyz',1, (random()*26)::integer)
> Random string from a list of strings
    (ARRAY['one','two','three'])[1+random()*3]
    
#ORDER
SELECT generate_series(1,10) as key, (random()*100.0)::numeric(4,2), 
       repeat ('1', (random() * 25)::integer );
   
#rul
   http://www.sqlmanager.net/products/postgresql/datagenerator
   http://www.datanamic.com/datagenerator/index.html
   
# SELECT count(*) FROM t_user  count where random() < 0.01 ;
```

#### dump

```
pg_dump –-exclude-table=MyBigTable > db.dmp
pg_dump –-table=MyBigTable –schema-only > mybigtable.schema

psql -c '\copy (SELECT * FROM MyBigTable  a where random() < 0.01 ) as/\

\COPY sample FROM sample.csv CSV HEADERpostgres=# SELECT * FROM sample; 
```



### Security

```
> grant privileges to role, grant role to user.  don't grant privileges to user
> at least two types of users: administrators and end-users. It is not a good idea to let ordinary users create or change database object defnitions
> there can also be a manager role, which can grant and revoke roles for other users, but is not supposed to do anything else.
```

#### Revoking user access to a table

```
revoke all on mysecrettable from username;
revoke all on mysecrettable from PUBLIC;
```

#### Best practices

```
> a good idea to include GRANT and REVOKE statements in the database creation script always

CREATE TABLE table1(...);
REVOKE ALL ON table1 FROM GROUP PUBLIC;
GRANT SELECT ON  table1 TO GROUP webreaders;
GRANT SELECT, INSERT, UPDATE, DELETE ON  table1 TO editors;
GRANT ALL ON table1 TO  admins;
```

#### Default search path

```
#psql
show search_path ;

#To see which table would be affected if you omit the schema name
\d tabname
```

#### Granting user access to a table

```
GRANT ALL ON someschema TO somerole;
GRANT SELECT, INSERT, UPDATE, DELETE ON  someschema.sometable TO somegroup;
GRANT somerole TO someuser, otheruser;

```

#### Access to schema is also needed

```
GRANT USAGE ON SCHEMA someschema TO someuser;
```

#### Granting access to a table through a group role

```
CREATE GROUP webreaders;
GRANT SELECT ON pages TO webreaders;
GRANT INSERT ON viewlog TO webreaders;
GRANT webreaders TO tim, bob;
```

#### Creating a new user

```
createuser bob
#psql
CREATE ROLE bob WITH NOSUPERUSER INHERIT NOCREATEROLE CREATEDB LOGIN;
CREATE ROLE tim WITH SUPERUSER;
#view user info
\du bob

#CREATE USER and CREATE GROUP are actually variations of CREATE ROLE
#CREATE USER  u == CREATE ROLE u LOGIN; 
#CREATE GROUP g == CREATE ROLE g NOLOGIN;


```

#### Update user privileges

```
#enable and disable user login
alter user bob nologin;
alter user bob login;

#Limiting concurrent connections 
alter user bob connection limit 0;
alter user bob connection limit 10;
alter user bob connection limit -1;
```

#### Forcing NOLOGIN users to disconnect

```
#psql, run as superuser 
SELECT pg_terminate_backend(procpid | pid) FROM from pg_stat_activity a   
  JOIN pg_roles r ON a.usename = r.rolname AND not rolcanlogin;
```

#### Removing a user without dropping their data

```
drop user bob;
#or just 
alter user bob nologin;
#or replace
grant bob bobs_replacement;
REASSIGN OWNED BY bob TO  bobs_replacement;
```

#### Checking all users have a secure password

```
#unencrypted
select usename,passwd from pg_shadow where passwd not like 'md5%' or length(passwd) <> 35;
#encrypted
select usename,passwd from pg_shadow where passwd like 'md5%' and length(passwd) = 35;

#passwordcheck - checks the password strength
http://developer.postgresql.org/pgdocs/postgres/passwordcheck.html
```

#### Giving  limited  superuser  powers  to  specific  users



```
ALTER ROLE BOB WITH CREATEDB;
ALTER ROLE BOB WITH CREATEUSER;

# superuser can write a special wrapper function
create or replace function copy_from(tablename text, filepath text)
returns void
security definer
as$$ 
...
;$$ language plpgsql;

revoke all on function copy_from( text,  text) from public;
grant execute on function copy_from( text,  text) to bob;

#SECURITY INVOKER | DEFINER
```

#### Auditing DDL changes

```
#vi postgresql.conf
log_statement = 'ddl'
#postgresql reload
egrep -i "create|alter|drop" \ /var/log/postgresql/postgresql-8.4-main.log
```



### Database  Administration

```
#sqls
DROP VIEW IF EXISTS cust_view;
CREATE OR REPLACE VIEW cust_view ASSELECT * FROM cust;

#cannot be uses transactions
> CREATE DATABASE / DROP DATABASE
> CREATE TABLESPACE / DROP TABLESPACE
> CREATE INDEX CONCURRENTLY
> VACUUM

# psql script that exits on first error and errno=3
\set ON_ERROR_STOP
....
psql -f test.sql   #work

#script file without "\set ON_ERROR_STOP" will not work for "-v ON_ERROR_STOP"
psql -f test.sql -v ON_ERROR_STOP

#save query result to local file
\t    #show only rows
\o filename.txt

#Adding/Removing the columns of a table
ALTER TABLE mytable ADD COLUMN last_update_timestamp TIMESTAMP WITHOUT TIME ZONE;
ALTER TABLE mytable DROP COLUMN last_update_timestamp;
#combine multiple operations
ALTER TABLE mytable DROP COLUMN IF EXISTS last_update_timestamp,ADD COLUMN last_update_timestamp TIMESTAMP WITHOUT TIME ZONE; 

ALTER TABLE x  DROP COLUMN  last_update_timestamp CASCADE;

#Adding/Removing schemas
CREATE SCHEMA schemaname;
CREATE SCHEMA schemaname AUTHORIZATION username;
CREATE SCHEMA AUTHORIZATION username;  #create schema username, owner username
DROP SCHEMA str;
DROP SCHEMA IF EXISTS newschema;

ALTER TABLE cust SET SCHEMA anotherschema;
ALTER SCHEMA existingschema RENAME TO anotherschema;

#tablespace
> Be empty
> Be owned by the PostgreSQL owning userid
> Be specifed with an absolute path name
CREATE TABLESPACE new_tablespace LOCATION '/usr/local/pgsql/new_tablespace';
DROP TABLESPACE new_tablespace;
ALTER TABLESPACE new_tablespace OWNER TO eliza;
ALTER USER eliza SET default_tablespace = 'new_tablespace';
#Tablespace-level tuning
ALTER TABLESPACE new_tablespace SET  (seq_page_cost = 0.05, random_page_cost = 0.1);
ALTER TABLE mytable SET TABLESPACE new_tablespace;
ALTER INDEX mytable_val_idx SET TABLESPACE new_tablespace;

#Accessing objects in other PostgreSQL databases
dblink

#Making views updateable  :  RULE
CREATE RULE cust_view_insert AS 
ON insert TO cust_view
DO INSTEAD
INSERT INTO cust VALUES (new.customerid, new.firstname, new.lastname, new.age);

```



### Backup & Recovery

If PostgreSQL crashes there will be a message in the server log with severity-level PANIC. PostgreSQL will immediately restart and attempt to recover using the transaction log or Write Ahead Log (WAL).

```
> logical backup—using pg_dump
pg_dumpall -g
pg_dump -F c > dumpfile
pg_dump –F c –f dumpfile

> physical backup—fle system backup
#gpdb NOTICE:  pg_stop_backup() is not supported in Greenplum Database
#pg 8
SELECT pg_start_backup('label');
SELECT pg_stop_backup();
#pg 10.0
SELECT pg_start_backup('label', false, false);
SELECT * FROM pg_stop_backup(false, true);
#https://www.postgresql.org/docs/10/static/continuous-archiving.html#BACKUP-BASE-BACKUP

```

#### Hot  logical  backup  of  all  tables  in  a  tablespace

```
SELECT 'pg_dump' 
UNION ALL
SELECT '-t ' || spcname || '.' || relname
FROM pg_class t 
   JOIN pg_tablespace ts ON reltablespace = ts.oid AND spcname = :TSNAME 
   JOIN pg_namespace n ON n.oid = t.relnamespace
WHERE relkind = 'r'
UNION ALL
SELECT '-F c > dumpfile';  
-- dumpfile is the name of the output fileExecute the query to build the pg_dump sc
```

