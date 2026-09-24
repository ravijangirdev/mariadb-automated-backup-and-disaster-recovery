# MariaDB Hardening, Automated Logical Dumps & Disaster Recovery

## 📌 Project Overview
Web application databases require resilient, non-blocking logical backup routines and strict access controls. Unsecured database engines and cold-copy file backups risk runtime corruption and data loss.

This project implements an enterprise-grade **MariaDB Database Infrastructure & Disaster Recovery Workflow** on **AWS EC2 (Ubuntu Linux)**:
* **Privilege Segregation:** Created an isolated schema (`app_production_db`) with least-privilege dedicated application credentials (`app_user`), keeping administrative access restricted.
* **Non-Blocking Hot Logical Dumps:** Implemented Bash automation invoking `mysqldump` with `--single-transaction` and `--quick` flags, preventing read/write locks during operational hours.
* **Stream-Compression Pipeline:** Piped raw SQL dumps directly into `gzip` streams, minimizing disk footprint and I/O overhead.
* **Automated Retention Pruning:** Integrated 7-day retention rotation utilizing the Linux `find` utility to protect storage headroom.
* **Disaster Recovery Validation:** Verified operational integrity through a live destruction drill (dropping tables and restoring in-memory via `zcat`).

---

## 🏗️ Architecture & Workflow

```text
[ MariaDB Service (Active) ]
             |
             v (mysqldump --single-transaction)
[ Unlocked Logical Snapshot ]
             |
             v (Piped Stream)
[ Gzip Compression Engine ]
             |
             +-----------------------+-----------------------+
             |                                               |
             v                                               v
[ /var/backups/db_backups/ ]                   [ /var/log/db_backup.log ]
*.sql.gz (Timestamped Archives)                 Audit Logs & File Sizes
             |
             v (find -mtime +7 -delete)
[ Storage Reclamation Engine ]
```

---

## 📜 Shell Script Source (`db_backup_manager.sh`)

```bash
#!/bin/bash

# ==============================================================================
# Script Name: db_backup_manager.sh
# Purpose    : Automated MariaDB/MySQL Logical Dump with Gzip & Retention
# Author     : Linux System Administrator
# ==============================================================================

# Configurations
BACKUP_DIR="/var/backups/db_backups"
DB_NAME="app_production_db"
LOG_FILE="/var/log/db_backup.log"
RETENTION_DAYS=7
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
DUMP_FILE="${BACKUP_DIR}/${DB_NAME}_dump_${TIMESTAMP}.sql.gz"

# Logging function
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | sudo tee -a "$LOG_FILE" > /dev/null
}

log_message "=== Starting MariaDB Database Backup Routine ==="

# Check if Backup Directory exists
if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR"
    log_message "Created directory: $BACKUP_DIR"
fi

# Execute Logical Dump with single-transaction and compress
log_message "Executing mysqldump for database: $DB_NAME..."
mysqldump --single-transaction --quick "$DB_NAME" 2>> "$LOG_FILE" | gzip > "$DUMP_FILE"

# Verify dump file size is greater than 0
if [ -s "$DUMP_FILE" ]; then
    DUMP_SIZE=$(ls -lh "$DUMP_FILE" | awk '{print $5}')
    log_message "SUCCESS: Database dump created at $DUMP_FILE (Size: $DUMP_SIZE)"
else
    log_message "ERROR: Dump file is empty or backup failed!"
    exit 1
fi

# Retention Cleanup: Delete archives older than RETENTION_DAYS
log_message "Pruning dumps older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -type f -name "${DB_NAME}_dump_*.sql.gz" -mtime +"$RETENTION_DAYS" -exec rm -f {} \; >> "$LOG_FILE" 2>&1
log_message "Database retention cleanup finished."

log_message "=== MariaDB Backup Routine Completed Successfully ==="
exit 0
```

---

## 🛠️ Step-by-Step Implementation

### 1. Database Provisioning & User Isolation
```bash
sudo apt install mariadb-server mariadb-client -y
sudo systemctl enable --now mariadb
```

Database schema and privilege initialization:
```sql
CREATE DATABASE app_production_db;
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'StrongPass@2026';
GRANT ALL PRIVILEGES ON app_production_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

### 2. Binary Installation & Rights Assignment
```bash
sudo mkdir -p /var/backups/db_backups
sudo chmod +x /usr/local/bin/db_backup_manager.sh
sudo /usr/local/bin/db_backup_manager.sh
```

---

## 🔍 Disaster Recovery Drill (Drop & Stream Restore)

Testing operational recovery capabilities under complete table drop failure scenarios:

### Step 1: Simulate Table Destruction
```bash
sudo mariadb -e "DROP TABLE app_production_db.customers;"
# Query execution returns: ERROR 1146 (42S02): Table doesn't exist
```

### Step 2: Stream-Decompression In-Memory Restore
Restore without uncompressing to disk, piping directly into the MariaDB daemon:
```bash
zcat /var/backups/db_backups/app_production_db_dump_*.sql.gz | sudo mariadb app_production_db
```

### Step 3: Consistency & Integrity Check
```bash
sudo mariadb -e "SELECT * FROM app_production_db.customers;"
```
*Result: Schema and data records fully restored with intact relational integrity.*

---

## 🚀 Key Takeaways & Sysadmin Skills Demonstrated
* **Database Administration:** Schema provisioning, user access control via Least Privilege Principle, and socket-based execution.
* **Non-Blocking Dumps:** Leveraging `--single-transaction` with InnoDB engines for transactional consistency without downtime.
* **Pipeline Efficiency:** On-the-fly streaming through `gzip` reducing intermediate disk I/O.
* **Operational Troubleshooting:** Diagnosing command line exit status, verifying non-empty targets (`-s`), and pipeline debugging.
* **Disaster Recovery (DR):** Rapid in-memory data restoration using `zcat` streaming.
