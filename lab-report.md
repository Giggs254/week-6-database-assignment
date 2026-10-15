# PostgreSQL Backup, Recovery, and Streaming Replication Lab Report

**Student:** Ian Giggs Okochi  
**Database System:** PostgreSQL 18  
**Operating System:** Ubuntu Linux  
**Database:** `bootcamp`  
**Repository:** [week-6-database-assignment](https://github.com/Giggs254/week-6-database-assignment)

---

## 1. Introduction

This practical lab demonstrates essential PostgreSQL database administration techniques, including logical backup and restoration, Write-Ahead Log (WAL) archiving, physical base backups, Point-in-Time Recovery (PITR), streaming replication, and replication monitoring.

The exercises were conducted using PostgreSQL 18 on Ubuntu Linux. Separate data directories were used for recovery and standby testing to protect the original database.

## 2. Objectives

The objectives of this practical were to:

1. Create a logical database backup and verify that it could be restored.
2. Configure WAL archiving and create a physical base backup.
3. Demonstrate Point-in-Time Recovery using a named restore point.
4. Configure a standby PostgreSQL server using streaming replication.
5. Monitor replication status and verify that data changes on the primary reached the standby.

## 3. Task One: Logical Backup and Restore

### 3.1 Procedure

Created a directory for the logical backup:

```bash
mkdir -p ~/backups
```

Created a custom-format backup of the `bootcamp` database:

```bash
pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp
```

Listed the backup contents:

```bash
pg_restore --list ~/backups/bootcamp.dump
```

Created a separate database for restoration testing and restored the backup:

```bash
createdb bootcamp_check
pg_restore -d bootcamp_check ~/backups/bootcamp.dump
```

Verified the restored tables:

```sql
\dt
```

### 3.2 Results

- The logical backup was created successfully.
- The backup contents were inspected using `pg_restore --list`.
- The backup was restored into the separate `bootcamp_check` database.
- The `public.students` table was present in the restored database.

**Conclusion:** Logical backup and restoration were demonstrated successfully without replacing the original database.

## 4. Task Two: WAL Archiving and Physical Base Backup

### 4.1 WAL Configuration

The PostgreSQL configuration was updated to enable WAL archiving.

Relevant settings:

```conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
```

The archive directory was:

```text
/var/lib/postgresql/wal_archive/
```

Verified the configuration using:

```sql
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
```

### 4.2 Physical Base Backup

Created a directory for the physical backup and ran `pg_basebackup`:

```bash
sudo install -d -o postgres -g postgres -m 700 /var/lib/postgresql/backups/base
```

```bash
sudo -u postgres pg_basebackup \
  -D /var/lib/postgresql/backups/base \
  -Ft -z -Xs -P
```

The backup produced the following files:

- `base.tar.gz` — compressed physical database backup.
- `pg_wal.tar.gz` — WAL files included with the backup.
- `backup_manifest` — metadata used to verify the backup's contents.

### 4.3 Results

- WAL archiving was enabled.
- Archived WAL files were generated.
- The archive statistics showed zero failed archive operations at the time of verification.
- The physical base backup was created successfully.

**Conclusion:** WAL archiving and physical backup creation were demonstrated successfully.

## 5. Task Three: Point-in-Time Recovery (PITR)

### 5.1 Preparation

Created a test table in the original `bootcamp` database:

```sql
CREATE TABLE pitr_demo (
    id integer PRIMARY KEY,
    note text
);

INSERT INTO pitr_demo
VALUES (1, 'Restore point recovery test');
```

Created a named restore point before simulating data loss:

```sql
SELECT pg_create_restore_point('pitr_before_drop');
```

The restore point was recorded at WAL location `0/602C110`.

Simulated data loss by dropping only the test table:

```sql
DROP TABLE pitr_demo;
```

The original `students` table was not dropped.

### 5.2 Recovery Procedure

A separate recovery data directory was prepared from the physical base backup. A dedicated PostgreSQL configuration was used with the recovery settings:

```conf
data_directory = '/var/lib/postgresql/18/pitr-test'
listen_addresses = '127.0.0.1'
port = 55432
unix_socket_directories = '/tmp'

archive_mode = off
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'

recovery_target_name = 'pitr_before_drop'
recovery_target_action = 'pause'
hot_standby = on
```

The recovery instance was started separately from the primary server.

### 5.3 Verification

Queried the recovered table on the recovery instance:

```bash
sudo -u postgres psql \
  -h /tmp -p 55432 -d bootcamp \
  -c "SELECT * FROM pitr_demo;"
```

The recovered result was:

```text
 id |           note
----+---------------------------
  1 | Restore point recovery test
```

### 5.4 Results

- Recovery stopped at the named restore point `pitr_before_drop`.
- The `pitr_demo` table and its test record were available in the recovered instance.
- Recovery was performed in a separate data directory, protecting the original database.

**Conclusion:** Point-in-Time Recovery to a named restore point was demonstrated successfully.

## 6. Task Four: Streaming Replication

### 6.1 Replication Role

Created a dedicated replication account:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '<YOUR_LAB_PASSWORD>';
```

The password placeholder represents a locally configured secret. The actual password must not be committed to this repository.

Added an appropriate replication access rule to `pg_hba.conf`:

```conf
host    replication    replicator    127.0.0.1/32    scram-sha-256
```

Reloaded the primary PostgreSQL configuration:

```bash
sudo systemctl reload postgresql@18-main
```

### 6.2 Initialize the Standby

Created a separate standby data directory:

```bash
sudo install -d -o postgres -g postgres -m 700 /var/lib/postgresql/18/standby
```

Used `pg_basebackup` to initialize the standby and generate standby connection settings:

```bash
sudo -u postgres pg_basebackup \
  -h 127.0.0.1 -p 5432 \
  -U replicator \
  -D /var/lib/postgresql/18/standby \
  -R -P
```

The standby was configured to listen on port `55433`, using a dedicated configuration file and `hot_standby = on`.

### 6.3 Verify Standby Status

Ran:

```sql
SELECT pg_is_in_recovery();
```

The query returned:

```text
t
```

This confirmed that the standby was operating in recovery mode.

### 6.4 Test Replication

Created a demonstration table on the primary:

```sql
CREATE TABLE IF NOT EXISTS replication_demo (
    id integer PRIMARY KEY,
    note text
);
```

Inserted a test record:

```sql
INSERT INTO replication_demo
VALUES (1, 'Streaming replication works');
```

Queried the standby:

```bash
sudo -u postgres psql \
  -h /tmp -p 55433 -d bootcamp \
  -c "SELECT * FROM replication_demo;"
```

The standby returned the record:

```text
 id |             note
----+-------------------------------
  1 | Streaming replication works
```

**Conclusion:** Streaming replication was demonstrated successfully, and a change made on the primary was visible on the standby.

## 7. Task Five: Monitor Replication Health

### 7.1 Primary Monitoring

Queried the primary's replication statistics:

```sql
SELECT
    application_name,
    state,
    sync_state,
    sent_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Measured the observed WAL difference:

```sql
SELECT
    application_name,
    state,
    pg_size_pretty(
        pg_wal_lsn_diff(sent_lsn, replay_lsn)::bigint
    ) AS lag_bytes
FROM pg_stat_replication;
```

### 7.2 Standby Monitoring

Checked the standby recovery status:

```sql
SELECT pg_is_in_recovery();
```

Checked the standby's WAL receiver:

```sql
SELECT
    status,
    sender_host,
    sender_port,
    written_lsn,
    flushed_lsn,
    latest_end_lsn
FROM pg_stat_wal_receiver;
```

### 7.3 Results

The observed replication status was:

| Metric | Observed result |
|---|---|
| Standby recovery status | `true` |
| Replication state | `streaming` |
| Observed replication lag | `0 bytes` |
| Test record on standby | Present |

These results represent the state observed during the practical, not a guarantee of zero lag at all times.

**Conclusion:** PostgreSQL replication monitoring views were used to verify the streaming connection and the replication of test data.

## 8. Summary of Results

| Task | Result |
|---|---|
| Logical backup | Completed |
| Logical restore verification | Completed |
| WAL archiving | Enabled and verified |
| Physical base backup | Created |
| Named-restore-point PITR | Successful |
| Streaming standby | Operational |
| Replication health monitoring | Verified |

## 9. Security and Best Practices

1. Store database backups securely and restrict access to backup directories.
2. Do not commit passwords, authentication secrets, or sensitive database records to GitHub.
3. Test restoration regularly instead of assuming a backup is usable.
4. Monitor WAL archive failures and replication lag.
5. Keep recovery and standby instances isolated from the primary data directory during testing.
6. Store large database backup archives outside the source-code repository unless explicitly required by the assignment.

## 10. Conclusion

The practical successfully demonstrated PostgreSQL backup and recovery techniques, including logical backups, physical base backups, WAL archiving, named-restore-point recovery, streaming replication, and replication monitoring.

The restoration and recovery exercises verified that database objects could be recovered, while the streaming replication test confirmed that data changes on the primary were available on the standby.

These procedures provide a foundation for database protection, recovery planning, and high-availability administration.

---

**End of Lab Report**
