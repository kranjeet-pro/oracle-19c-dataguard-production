# architecture.md

## Oracle Data Guard Architecture (Primary → Standby)

### Overview

Oracle Data Guard provides **high availability, data protection, and disaster recovery** by maintaining a synchronized standby database.

This setup uses **Physical Standby** for maximum data safety and performance.

---

### Logical Flow

```
Client Connections
        |
        v
+-------------------+
|  PRIMARY DATABASE |
|  ORCL (CDB)       |
|  PDB: CBSPDB      |
+-------------------+
        |
        |  Redo Transport (ARCH / LGWR)
        v
+-------------------+
|  STANDBY DATABASE |
|  ORCLSTBY (CDB)   |
|  Managed Recovery |
+-------------------+
```

---

### Key Components

| Component    | Description                      |
| ------------ | -------------------------------- |
| Primary DB   | Production database (read/write) |
| Standby DB   | Physical replica (recovery mode) |
| Redo Logs    | Changes captured on Primary      |
| Archive Logs | Shipped to Standby               |
| MRP          | Managed Recovery Process         |

---

### Data Flow (Step-by-Step)

1. User commits transaction on **Primary DB**
2. Redo generated in **Online Redo Logs**
3. Redo archived (ARCH)
4. Redo shipped to Standby via **redo transport service**
5. Standby applies redo using **MRP process**
6. Standby remains in sync with Primary

---

### Protection Mode Used

```
Maximum Performance (ASYNC)
```

* Best balance of performance & safety
* Minimal impact on Primary workload

---

### Switchover & Failover

| Operation  | Description                            |
| ---------- | -------------------------------------- |
| Switchover | Planned role reversal (zero data loss) |
| Failover   | Unplanned recovery during outage       |

---

### Validation Queries

```sql
SELECT DATABASE_ROLE, OPEN_MODE FROM V$DATABASE;
SELECT NAME, VALUE FROM V$DATAGUARD_STATS;
```

---

# prerequisites.md

## System Prerequisites (Primary & Standby)

### OS Requirements

| Item         | Value                 |
| ------------ | --------------------- |
| OS           | Rocky Linux 8.10      |
| Architecture | x86_64                |
| SELinux      | Permissive / Disabled |
| Time Sync    | chronyd enabled       |

---

### Required Packages

```bash
dnf install -y bc binutils elfutils-libelf gcc gcc-c++ glibc \
libaio libXrender libX11 libXau libXi libXtst \
make sysstat unixODBC ksh
```

---

### Kernel Parameters

Add to `/etc/sysctl.conf`:

```ini
fs.file-max = 6815744
kernel.sem = 250 64000 100 4096
kernel.shmmni = 8192
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
net.ipv4.ip_local_port_range = 9000 65535
vm.swappiness = 1
```

Apply:

```bash
sysctl -p
```

---

### User Limits

Add to `/etc/security/limits.conf`:

```ini
oracle soft nofile 65536
oracle hard nofile 65536
oracle soft nproc 16384
oracle hard nproc 16384
```

---

### Filesystem Layout

```
/orcl    → Oracle binaries + data
/u01     → Optional backups
```

---

### Network & Firewall

```bash
firewall-cmd --permanent --add-port=1521/tcp
firewall-cmd --reload
```

---

### Oracle Environment Variables

```bash
export ORACLE_BASE=/orcl/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19c/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
```

---

# troubleshooting.md

## Common Oracle Data Guard Issues & Fixes

---

### ORA-01034: ORACLE not available

**Cause:** Instance not started

**Fix:**

```bash
sqlplus / as sysdba
startup
```

---

### ORA-00205: error in identifying control file

**Cause:** Wrong control_files path

**Fix:**

```sql
SHOW PARAMETER control_files;
```

Verify files exist and permissions are correct.

---

### Standby Not Applying Redo

**Check:**

```sql
SELECT PROCESS, STATUS FROM V$MANAGED_STANDBY;
```

**Fix:**

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

---

### Redo Transport Error

**Check:**

```sql
SELECT DEST_ID, STATUS, ERROR FROM V$ARCHIVE_DEST;
```

**Fix:**

* Verify TNS entries
* Check listener on Standby
* Ensure firewall allows port 1521

---

### PDB Not Opening After Restart

**Fix:**

```sql
ALTER PLUGGABLE DATABASE ALL OPEN;
ALTER PLUGGABLE DATABASE ALL SAVE STATE;
```

---

### Archive Log Space Full

**Fix:**

```sql
RMAN> DELETE ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-2';
```

---

### Useful Monitoring Queries

```sql
SELECT NAME, VALUE FROM V$DATAGUARD_STATS;
SELECT SEQUENCE#, APPLIED FROM V$ARCHIVED_LOG ORDER BY SEQUENCE# DESC;
```

---

## Final Tip

Always monitor:

* Archive log generation
* Apply lag
* Disk usage on `/orcl`

Data Guard is only as strong as its monitoring.
