
======================
pg_dump and pg_dumpall
Restore with psql
https://www.postgresql.org/docs/current/app-pgdump.html
======================

1. Drop and create the test table
---------------------------------
* Open a psql session and run the SQLs

-- Create a database buptest
CREATE DATABASE buptest;

-- Connect to buptest
\c  buptest

-- Create the table
CREATE TABLE test (id  int);

-- Insert some rows
INSERT INTO test
SELECT * FROM generate_series(1, 10);

-- Quit
\q

2. Take a SQL dump
------------------
* Run this on bash terminal

pg_dump buptest  >   buptest.dump.sql

3. Check out the content of the sql dump file
---------------------------------------------

* open the file in editor such as vi or nano
vi buptest.dump.sql

4. Recreate the test database
-----------------------------

* Drop and recreate the buptest database

-- Drop
psql -c "DROP DATABASE  buptest;"

-- Create 
psql -c "CREATE DATABASE buptest;"


4. Restore test database
------------------------

psql  -d buptest  < buptest.dump.sql

5. Checkout the data in the test table
--------------------------------------

psql -d  buptest

SELECT * FROM test;

======================================
pg_restore
Requires pg_dump file in custom format
https://www.postgresql.org/docs/current/app-pgrestore.html
======================================

1. Dump data in custom format
-----------------------------

* Dump in pgdump custom format
* Dump in a directory with -F d

pg_dump -F c   buptest  >   buptest.dump.custom

2. Drop and Recreate the test database
--------------------------------------
* These are for demo only as same effect may be achieved with -c and -C options
psql -c "DROP DATABASE buptest;"

psql -c "CREATE DATABASE buptest;"


3. Restore the database from dump
---------------------------------
* Creates the DB objects and populate
pg_restore   -d buptest < buptest.dump.custom

* Drops the database recreates it
pg_restore  -d postgres -c -C <  buptest.dump.custom

4. Clean up
-----------

* You may drop the test database once you are done with this exercise

psql -c "DROP DATABASE buptest;"


==========
pg_dumpall
==========

1. Create SQL file with globals
-------------------------------
* Passwords cannot be read in RDS/Aurora so option is neccesary

pg_dumpall --no-role-passwords  --globals-only > globals.sql

* The generated file may be edited to add passwords/etc

2. Create user to be used with pg_dumpall
-----------------------------------------
* Requires a user that has appropriate access to read all datbases
CREATE ROLE user1  LOGIN PASSWORD 'user1';
GRANT rds_superuser TO user1;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO user1;

pg_dumpall --no-role-passwords   > dumpall.sql

========================
Change the backup window
========================

1. Open the RDS console
-----------------------

* Select the DB cluster
* Click on Modify


2. Get the current UTC time
---------------------------

* Open google.com
* Search for   *current UTC time*

3. Modify the cluster
---------------------
* Set the automatic backup time = UTC Time from #2  + 5 minutes
* Select Apply Immediately

4. Wait for 5 minutes
---------------------
* You should see the automatic backup

===============================
Restore a cluster from snapshot
===============================

1. Cluster restore from automatic snapshot
------------------------------------------
* In RDS console click on 'snapshot' in left navigation panel
* Select the system or manual snapshot
* Actions >> Restore snapshot

2. Set the configuration for the restored cluster
-------------------------------------------------

* DB Cluster Identifier = rdsa-postgresql-restored

* Verify Connectivity parameters = Go with default for the existing DB cluster
    - RDSA VPC
    - Subnet Group  (rdsa-postgresql-db-subnet-group)
    - Security Group  (rdsa-security-group-internal)

* Instance class
    -  Select Burstable instance db.t3.medium

* Restore Database

3. Check availability of the restored cluster
---------------------------------------------

* Open a Bastion host session

* Check the value for LatestRestorableTime & EarliestRestorableTime

aws rds describe-db-clusters \
    --db-cluster-id  rdsa-postgresql-restored-cluster  \
    --query 'DBClusters[0].{Latest:LatestRestorableTime,Earliest:EarliestRestorableTime}'

* If you get NULL for the two parameters then cluster is still creating

* Note down the 'LatestRestorableTime'

4. Test out the cluster
-----------------------
* Start a psql session
* Get the Cluster Endpoint for the restored cluster
* Use psql to test

psql -h <<Paste cluster endpoint>>

* INSERT a couple of rows in the test table

5. Check the LatestRestorableTime
---------------------------------

* May have to check multiple times
* After few minutes, you will see a change in 'LatestRestorableTime'


Cleanup
-------
* Delete the instance in new cluster


==============================================
Restore a cluster from existing cluster (PITR)
==============================================

aws rds describe-db-clusters \
    --db-cluster-id  rdsa-postgresql-cluster  \
    --query 'DBClusters[0].{Latest:LatestRestorableTime,Earliest:EarliestRestorableTime}'

1. Recreate the labdb=>test table and populate with some data
-------------------------------------------------------------

-- Truncate table
TRUNCATE TABLE test;

-- Populate table with 5 rows
INSERT INTO test
SELECT * FROM generate_series(1, 5);

-- Time-1: Note down the UTC time
\! date

2. Wait for 30 seconds and insert another 5 Rows
-------------------------------------------------

INSERT INTO test
SELECT * FROM generate_series(6, 10);

-- Time-2: Note down the UTC time
\! date

3. Wait for 30 seconds and insert another 5 Rows
-------------------------------------------------

INSERT INTO test
SELECT * FROM generate_series(11, 15);

-- Time-3: Note down the UTC time
\! date

-- Checkout the data
SELECT count(*) FROM test;

4. Restore the cluster to a state prior to last insert
------------------------------------------------------
* Select the existing DB cluster
* Actions >> Restore to point in time

* Set
    - Custom date time = Batch#2 time converted to local timezone
    - DB Cluster Identifier = rdsa-postgresql-pitr

5. Test the restored cluster
----------------------------
* Use psql to connect to the restored cluster
psql -h <<paste the cluster endpoint for restored cluster>>

* Check the data in test table
SELECT * FROM test;

* You should see rows with [id=1, 10]


====================
Cross Region copying
https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_CopySnapshot.html
====================


============================
Export snapshot to S3 bucket
============================

Exercise setup
--------------
* We will be using utility scripts under the subfolder
  ~/Amazon-RDS-Aurora-Postgres-v1/backup



1. Create an S3 bucket
----------------------
* Create OR you may use the existing bucket
* Bucket should be in the *SAME region as the snapshot*

My bucket name = rdsa-copy-to-s3

Note: Bucket names are unique

2. Create a KMS key
-------------------
* Open the KMS console
* On left navigation panel select
  'Customer Managed Key'

* Select 'Symmetric Key'

* Name = rdsa-cmk-for-s3-export

3. Create the role
------------------
* Use the utility script - pass bucket as the parameter
* Change directory & chmod for the utility scripts

cd ~/Amazon-RDS-Aurora-Postgres-v1/backup
chmod u+x *.sh

./setup-s3-copy-task-role.sh     <<Your bucket name>>


4. Use console to export the snapshot
-------------------------------------
* This may be done with CLI as well

* Select the snapshot
* Actions >> Export to S3

* Export identifier = rdsa-s3-export-test
* Amount of data = All
* S3 destination = <<Your bucket>>
* S3 destination prefix = <<optional>>
* IAM Role = rdsa-copy-to-s3-role
* Encryption = rdsa-cmk-for-s3-export

Cleanup steps
=============

1. Delete KMS Key
-----------------
* Open the KMS console - select 'Customer managed key' in left panel

* Select the key - rdsa-cmk-for-s3-export

* Schedule for deletion
  Actions >> Schedule for deletion
  Change waiting period = 7 days (minimum)

2. Empty & Delete S3 
----------------------
* Optional if you want to retain data & bucket

* Delete the s3 export folder on S3

* Delete the S3 bucket (if you no longer need it)

3. Delete the IAM role
----------------------
* Use the utility script to delete the role

cd ~/Amazon-RDS-Aurora-Postgres-v1/backup
./cleanup-s3-copy-task-role.sh

==========
AWS Backup
==========
https://docs.aws.amazon.com/aws-backup/latest/devguide/how-it-works.html

export PS1='$(pwd) $ '