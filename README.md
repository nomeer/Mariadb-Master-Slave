# MariaDB Master-Slave Replication Cluster

## 📋 Project Overview

This repository documents the architecture and manual configuration of a 3-node MariaDB High-Availability (HA) database cluster using asynchronous replication.

---

## 🗺️ Network Architecture

| Role | Node | IP Address |
|------|------|------------|
| Master / Write | Node 1 | `192.30.72.233` |
| Slave 01 / Read | Node 2 | `192.30.72.234` |
| Slave 02 / Read | Node 3 | `192.30.72.235` |

> **SSH Port:** `2229` (customized for security hardening)

---

## 🛠️ Step 1: Master Configuration (Node 1)

The Master node handles all database writes and records every change into a Binary Log so the slaves can replicate them.

**File: `/etc/mysql/mariadb.conf.d/50-server.cnf`**

```ini
[mysqld]
server_id = 1
log_bin = /var/log/mysql/mariadb-bin
```

**Creating the Replication User:**

This grants the slaves the specific permission to read the Master's binary log.

```sql
CREATE USER 'replica_user'@'%' IDENTIFIED BY 'StrongPassword123';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;

SHOW MASTER STATUS;
```

> The output of `SHOW MASTER STATUS` provides the exact log file and position needed for the slaves to begin syncing.

---

## ⚙️ Step 2: Slave Configuration (Nodes 2 & 3)

The slaves are configured to pull data from the Master and are locked to Read-Only to prevent accidental data corruption.

**File: `/etc/mysql/mariadb.conf.d/50-server.cnf`**

```ini
[mysqld]
server_id = 2  # Use 3 for Node 3
read_only = 1
```

**Connecting to the Master:**

```sql
CHANGE MASTER TO
  MASTER_HOST='192.30.72.233',
  MASTER_USER='replica_user',
  MASTER_PASSWORD='StrongPassword123',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mariadb-bin.000001',
  MASTER_LOG_POS=1234;

START SLAVE;
```

---

## 🔍 Step 3: Verifying Cluster Health

Run the following command on each slave to confirm replication is healthy:

```sql
SHOW SLAVE STATUS\G
```

**Critical metrics to verify:**

| Metric | Expected Value | Meaning |
|--------|---------------|---------|
| `Slave_IO_Running` | `Yes` | Successfully connected to the Master |
| `Slave_SQL_Running` | `Yes` | Successfully executing replicated commands |

---

## 🧠 Key Learnings & Security

### The Superuser Paradox

Setting `read_only = 1` successfully blocks standard users from writing to the slave databases. However, it does **not** block the `root` user or any user holding the `SUPER` privilege.

True cluster security requires provisioning dedicated, restricted application users that only possess `SELECT` permissions:

```sql
CREATE USER 'app_readonly'@'%' IDENTIFIED BY 'your_password_here';
GRANT SELECT ON *.* TO 'app_readonly'@'%';
FLUSH PRIVILEGES;
```

This ensures that even if `read_only` were accidentally disabled, the application user is physically incapable of writing data.
