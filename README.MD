
# Azure SQL Server Backup & Point-in-Time Recovery Lab

## Overview

This lab demonstrates how to protect a SQL Server database running on an Azure Virtual Machine using **Azure Backup and a Recovery Services Vault**.

The goal was not simply to configure a backup policy, but to perform and validate a complete recovery workflow.

The lab included:

- SQL Server workload discovery
- Azure Recovery Services Vault
- SQL database protection
- Full backups
- Differential backup configuration
- Transaction log backups
- Backup retention
- Backup monitoring
- Full database recovery
- Alternate-location restore
- Point-in-Time Recovery (PITR)
- Simulated accidental data deletion
- Recovery validation using SQL Server Management Studio (SSMS)

The protected application database used throughout the lab was:

`IqrasCafeDB`

---

# Architecture

```text
                    Microsoft Azure
                           |
                           |
                 +------------------+
                 |      SQL01       |
                 |    Azure VM      |
                 |                  |
                 |   SQL Server     |
                 +--------+---------+
                          |
                          |
                    IqrasCafeDB
                          |
                          |
               Azure Backup Extension
                          |
                          v
              +-----------------------+
              | Recovery Services     |
              | Vault                 |
              |                       |
              | RSV-SQL-Backup-Lab    |
              +-----------+-----------+
                          |
          +---------------+----------------+
          |               |                |
          v               v                v
     Full Backup    Differential       Transaction
                       Backup          Log Backups
                                         |
                                    Every 15 Minutes
                                         |
                                         v
                                Point-in-Time Recovery
                                         |
                                         v
                            +--------------------------+
                            | Restored SQL Database    |
                            |                          |
                            | IqrasCafeDB_restored_*   |
                            +--------------------------+
```

---

# Lab Objectives

The primary objectives were to:

1. Configure Azure Backup for SQL Server running inside an Azure VM.
2. Discover SQL Server workloads from a Recovery Services Vault.
3. Protect the application database using a custom backup policy.
4. Perform an on-demand full backup.
5. Restore the database without overwriting production.
6. Validate the restored data.
7. Configure transaction-log protection.
8. Simulate accidental deletion of production data.
9. Perform Point-in-Time Recovery.
10. Verify that the deleted data was successfully recovered.

---

# Environment

| Component | Purpose |
|---|---|
| Azure Virtual Machine | Hosts SQL Server |
| SQL Server | Database engine |
| SQL Server Management Studio | Database administration and recovery validation |
| Recovery Services Vault | Central Azure Backup service |
| Azure Backup | Protects SQL workloads |
| IqrasCafeDB | Application database used for testing |

SQL Server instance:

```text
SQL01\MSSQLSERVER
```

Recovery Services Vault:

```text
RSV-SQL-Backup-Lab
```

Backup policy:

```text
SQL-Production-DR-Policy
```

---

# Test Database

The lab used a small fictional cafe database to simulate a production workload.

Two primary tables were created:

```text
Customers
Orders
```

Example customer records:

```text
CustomerID    CustomerName
--------------------------------
1             Ali Khan
2             Umer Ahmed
3             Usman Malik
```

Example orders included:

```text
Cappuccino
Chicken Sandwich
Latte
```

The data provided a simple way to verify whether backup and recovery operations were actually successful.

---

# SQL Recovery Model

The application database was verified to use the **FULL recovery model**.

```sql
SELECT
    name,
    recovery_model_desc
FROM sys.databases
WHERE name = 'IqrasCafeDB';
```

Result:

```text
IqrasCafeDB    FULL
```

The FULL recovery model is important when transaction-log backups and point-in-time recovery are required.

---

# Recovery Services Vault

A Recovery Services Vault was created:

```text
RSV-SQL-Backup-Lab
```

The vault was configured to discover SQL workloads running on the Azure VM.

Azure discovered the SQL Server instance and its databases.

The application database was then selected for protection.

---

# Backup Policy

A custom backup policy was created:

```text
SQL-Production-DR-Policy
```

## Full Backup

```text
Frequency: Weekly
Day: Sunday
Time: 3:00 AM
Retention: 4 weeks
```

## Differential Backup

```text
Frequency:
Monday
Tuesday
Wednesday
Thursday
Friday
Saturday

Time: 2:00 AM

Retention: 7 days
```

## Transaction Log Backup

```text
Frequency: Every 15 minutes
Retention: 15 days
```

This provides significantly smaller potential data loss than relying only on daily or weekly backups.

---

# Understanding the Backup Chain

The recovery strategy can be visualized as:

```text
FULL BACKUP
     |
     +-------------------------------+
     |                               |
     v                               v
Differential                    Transaction Logs
Backup                          Every 15 Minutes
     |                               |
     +---------------+---------------+
                     |
                     v
                Recovery
```

The full backup establishes a recovery baseline.

Differential backups contain changes relative to the relevant full backup.

Transaction-log backups provide much finer recovery granularity and enable Point-in-Time Recovery.

---

# Initial Full Backup

After configuring protection, an on-demand full backup was triggered.

Azure Backup displayed the operation under:

```text
Recovery Services Vault
        ↓
Backup Jobs
```

The job was monitored until its status changed to:

```text
Completed
```

The database backup item subsequently reported:

```text
Backup Status: Healthy
```

---

# Full Database Restore Test

The first recovery test verified that Azure could restore the complete database.

Instead of overwriting the original database, **Alternate location** was selected.

This created another database on the SQL Server.

Example:

```text
IqrasCafeDB
IqrasCafeDB_restored_9_26_2026_1350
```

This approach allowed the restored database to be validated without modifying the original database.

---

# Validate Full Restore

The restored database was selected in SSMS.

The following queries were executed:

```sql
SELECT DB_NAME() AS CurrentDatabase;

SELECT * FROM Customers;

SELECT * FROM Orders;
```

The restored database contained the expected original customer and order records.

This confirmed that the full database restore was successful.

---

# Point-in-Time Recovery Test

The second recovery test simulated a realistic production incident.

The objective was:

```text
Create Data
      ↓
Transaction Log Backup
      ↓
Accidental DELETE
      ↓
Point-in-Time Restore
      ↓
Recover Deleted Data
```

---

# Create Recovery Test Record

A new customer was inserted into the production database.

```sql
USE IqrasCafeDB;
GO

INSERT INTO Customers (CustomerName, Email)
VALUES ('PITR Test Customer', 'pitrtest@iqrascafe.com');
GO

SELECT GETDATE() AS TransactionTime;

SELECT * FROM Customers;
GO
```

The new record appeared as:

```text
CustomerID: 5
CustomerName: PITR Test Customer
```

The transaction time was also recorded for recovery testing.

---

# Transaction Log Backup

The backup policy was configured to capture SQL transaction logs every:

```text
15 minutes
```

After the next transaction-log backup completed, Azure displayed a newer restore point.

This demonstrated that Azure Backup was continuously extending the recoverable timeline beyond the original full backup.

Conceptually:

```text
Full Backup
2:03 PM
   |
   | transactions
   |
   +---- Customer #5 created
   |
   | Transaction Log Backup
   |
2:18 PM
```

At this point, Customer #5 was protected by the recovery chain.

---

# Simulated Production Incident

An accidental DELETE was then simulated.

Before performing the deletion, the current time and existing customer records were captured.

```sql
SELECT GETDATE() AS BeforeDeleteTime;

SELECT * FROM Customers;
GO
```

The database contained five customers.

The incident was then simulated:

```sql
SELECT GETDATE() AS IncidentTime;

DELETE FROM Customers
WHERE CustomerID = 5;

SELECT * FROM Customers;
GO
```

After the DELETE, only four customers remained.

```text
Customer 5: MISSING
```

This represented an accidental production data-loss incident.

---

# Point-in-Time Restore

Azure Backup was then used to restore the database to a point **before the DELETE operation**.

The restore was performed to an alternate database rather than overwriting the existing production database.

This created another restored database.

Example:

```text
IqrasCafeDB_restored_9_26_2026_1423
```

This provided a safe recovery copy for validation.

---

# Validate Point-in-Time Recovery

The restored database was selected in SSMS.

The following query was executed:

```sql
SELECT DB_NAME() AS CurrentDatabase;

SELECT * FROM Customers;
```

The result contained:

```text
CustomerID    CustomerName
-----------------------------------------
1             Ali Khan
2             Umer Ahmed
3             Usman Malik
4             Recovery Test Customer
5             PITR Test Customer
```

Customer #5 had successfully returned.

This confirmed that the transaction-log recovery chain successfully recovered the database to a point before the accidental deletion.

---

# Recovery Result

## Before Incident

```text
Customers = 5
```

## After Accidental DELETE

```text
Customers = 4
```

## After Point-in-Time Recovery

```text
Customers = 5
```

Therefore:

```text
Backup Chain        = Working
Transaction Logs    = Working
Point-in-Time Restore = Working
Deleted Data        = Recovered
```

---

# Production Recovery Strategy

The lab intentionally restored the database under another name instead of immediately overwriting production.

A safer production workflow is:

```text
Production Incident
        |
        v
Identify Recovery Point
        |
        v
Restore Separate Database
        |
        v
Validate Recovered Data
        |
        +------------------------+
        |                        |
        v                        v
Recover Required Records    Controlled DB Cutover
        |                        |
        +-----------+------------+
                    |
                    v
             Validate Application
                    |
                    v
              Resume Service
```

This reduces the risk of destroying additional production data during the recovery process.

---

# Monitoring and Alerts

Backup and restore operations were monitored through the Recovery Services Vault.

The lab reviewed:

```text
Backup Jobs
Backup Items
Recovery Points
Backup Alerts
Backup Health
```

A backup alert was also observed for a SQL system database (`model`).

The alert was separate from the successfully protected `IqrasCafeDB` workload and demonstrated the importance of reviewing backup health at the individual workload level rather than assuming the entire SQL instance is healthy.

---

# Key Concepts Learned

### Full Backup

Creates a complete recoverable baseline of the database.

### Differential Backup

Captures changes relative to the applicable full backup, reducing the amount of data required compared with another full backup.

### Transaction Log Backup

Captures SQL transaction-log records at frequent intervals and enables granular Point-in-Time Recovery.

### Recovery Point Objective (RPO)

RPO represents the maximum acceptable amount of data loss measured in time.

With transaction-log backups scheduled every 15 minutes, the backup design targets a relatively small data-loss window, although actual recoverability depends on successful log backups and the available recovery chain.

### Recovery Time Objective (RTO)

RTO represents the target amount of time required to restore service following an incident.

The lab demonstrated that recovery time includes more than simply starting the restore:

```text
Detect Incident
      ↓
Choose Recovery Point
      ↓
Restore Database
      ↓
Validate Data
      ↓
Application/DB Cutover
      ↓
Resume Service
```

---

# Important Lessons

One of the biggest lessons from this lab was:

> A successful backup job does not prove that data can actually be recovered.

Recovery must be tested.

This lab therefore validated both sides:

```text
BACKUP
   +
RESTORE
   +
DATA VALIDATION
```

Another important lesson was that transaction-log backups allow recovery to a much more precise point than relying only on periodic full backups.

Finally, restoring to an alternate database provides a safer method for validating recovered data before modifying or replacing a production workload.

---

# Skills Practiced

- Azure Backup
- Recovery Services Vault
- Azure SQL workload protection
- SQL Server on Azure VM
- Backup policies
- Backup retention
- Full backups
- Differential backups
- Transaction-log backups
- Point-in-Time Recovery
- Alternate-location restore
- Backup monitoring
- Backup alerts
- Recovery validation
- Basic SQL queries
- RPO/RTO concepts
- Disaster-recovery testing
- Production recovery planning

---

# Screenshots

Screenshots from the lab can be stored in:

```text
/screenshots
```

Suggested structure:

```text
screenshots/
├── 01-sql-vm-and-database.png
├── 02-recovery-services-vault.png
├── 03-sql-workload-discovery.png
├── 04-backup-policy.png
├── 05-full-backup-job.png
├── 06-backup-healthy.png
├── 07-original-database-records.png
├── 08-full-database-restore.png
├── 09-restored-database-validation.png
├── 10-full-recovery-model.png
├── 11-pitr-test-customer-created.png
├── 12-transaction-log-recovery-point.png
├── 13-customer-delete-incident.png
├── 14-pitr-restore-job.png
├── 15-pitr-restored-database.png
├── 16-five-customers-recovered.png
└── 17-backup-monitoring-alert.png
```

---

# Outcome

The lab successfully demonstrated an end-to-end Azure SQL Server backup and recovery workflow.

A SQL database running on an Azure VM was protected using Azure Backup, restored from a full backup, subjected to a simulated data-loss incident, and recovered using transaction-log-based Point-in-Time Recovery.

The final validation confirmed that the deliberately deleted customer record was present in the restored database.

**Lab Status: Completed ✅**

---

## Next Lab

The next SQL disaster-recovery lab will extend this design with a more production-oriented Azure networking architecture, including:

- Private networking
- Private Endpoint / Private Link
- Private DNS
- Restricted public access
- Network-level connectivity validation
- SQL disaster-recovery drill
- Point-in-Time Recovery
- Recovery validation
- Controlled recovery/cutover workflow

The focus will remain on **Azure Infrastructure, Networking, Backup, and Disaster Recovery** rather than deep SQL database administration.
