# PostgreSQL Backup, Recovery, and Replication Lab

## Overview

This project demonstrates PostgreSQL database administration using logical backups, physical backups, Write-Ahead Log (WAL) archiving, Point-in-Time Recovery (PITR), and streaming replication.

## Objectives

1. Create and restore a logical database backup using `pg_dump` and `pg_restore`.
2. Enable WAL archiving and create a physical base backup using `pg_basebackup`.
3. Demonstrate Point-in-Time Recovery using a named restore point.
4. Configure a PostgreSQL standby server for streaming replication.
5. Monitor replication health and verify that data changes on the primary are received by the standby.

## Environment

- Operating system: Ubuntu Linux
- Database: PostgreSQL 18
- Primary port: `5432`
- Standby port: `55433`
- PITR test instance port: `55432`
- Database: `bootcamp`
- Main table: `students`

## Task 1: Logical Backup and Restore

Created a custom-format backup of the `bootcamp` database.

```bash
pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp
```

Verified the backup contents and restored the backup into a separate test database named `bootcamp_check`.

```bash
pg_restore --list ~/backups/bootcamp.dump
```

**Result:** The restore was verified, and the `students` table was present in the restored database.

## Task 2: WAL Archiving and Physical Backup

Configured PostgreSQL with:

- `wal_level = replica`
- `archive_mode = on`
- `archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'`

Created a physical base backup using `pg_basebackup`.

The backup artifacts included:

- `base.tar.gz`
- `pg_wal.tar.gz`
- `backup_manifest`

**Result:** WAL archiving was operational, and the physical backup files were created successfully.

## Task 3: Point-in-Time Recovery

Created a test table called `pitr_demo` and inserted a sample record. A named restore point, `pitr_before_drop`, was created before dropping the test table from the original database.

Recovery was performed in a separate PostgreSQL data directory using the archived WAL files.

**Result:** The PITR test instance recovered to the named restore point, and the original test record was successfully retrieved. The original `students` table was not deleted.

## Task 4: Streaming Replication

Created a PostgreSQL replication role and used `pg_basebackup` with the `-R` option to initialize the standby.

The standby was configured to run on port `55433`.

Verified the standby recovery status using:

```sql
SELECT pg_is_in_recovery();
```

**Result:** The query returned `true`, confirming that the standby was operating in recovery mode.

## Task 5: Replication Monitoring

Created a `replication_demo` table on the primary database and inserted a sample record.

Verified that the record was available on the standby and checked replication status using PostgreSQL's replication statistics.

**Result:** The standby received the sample record, replication was streaming, and the observed replication lag was `0 bytes` at the time of verification.

## Conclusion

The lab successfully demonstrated logical backup and restoration, physical backup creation, WAL archiving, Point-in-Time Recovery, streaming replication, and replication monitoring using PostgreSQL 18.

The recovery and replication tests were conducted using separate PostgreSQL data directories to avoid replacing the original database.

## Security Note

Database passwords, private credentials, and sensitive production data must not be committed to a public repository. Large binary database backups should be stored securely outside Git unless the assignment explicitly requires them.
