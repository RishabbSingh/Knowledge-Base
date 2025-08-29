# MongoDB Atlas Backup & Restore – Practical Guide

> Proven commands for taking dumps (`mongodump`) and restoring (`mongorestore`) from Linux (Ubuntu/Debian or Amazon Linux 2) and Windows. Includes full DB, single DB/collection, compression, and safety flags.

---

## 1) Prerequisites
- **MongoDB Database Tools** installed (`mongodump`, `mongorestore`).
- **Atlas connection string (URI)** with user & password that has read (for dump) or read/write (for restore) permissions.
- Your **public IP** added to Atlas Network Access (IP allowlist).
- Enough local disk space for the dump.

---

## 2) Install MongoDB Database Tools

### A) Amazon Linux 2 (EC2)
```bash
# Add MongoDB repo for Amazon Linux 2
cat <<'EOF' | sudo tee /etc/yum.repos.d/mongodb-org-6.0.repo
[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc
EOF

sudo yum update -y
sudo yum install -y mongodb-database-tools

mongodump --version   # verify
```

### B) Ubuntu / Debian
```bash
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -sc)/mongodb-org/6.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

sudo apt-get update
sudo apt-get install -y mongodb-database-tools

mongodump --version   # verify
```

---

## 3) Take a Backup (Dump) with `mongodump`
> Dumps are **binary snapshots** (BSON files). You can back up *everything*, a single DB, or a single collection.

### A) Dump all databases
```bash
mongodump \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  --out=/path/to/backup/atlas-dump-$(date +%F_%H-%M)
```

### B) Dump a single database
```bash
mongodump \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/myDatabase" \
  --out=/path/to/backup/atlas-dump-$(date +%F_%H-%M)
```

### C) Dump a single collection
```bash
mongodump \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/myDatabase" \
  --collection myCollection \
  --out=/path/to/backup/atlas-dump-$(date +%F_%H-%M)
```

### D) Compress on the fly (smaller footprint)
```bash
mongodump --uri="mongodb+srv://<USER>:<PASS>@cluster0.../myDatabase" --archive | gzip > myDatabase-$(date +%F_%H-%M).archive.gz
```
- Use `--archive` to stream to a single file (great for S3 uploads or scp).

### E) Useful options
- `--numParallelCollections <N>` – speed up large dumps.
- `--query '{...}'` – dump subset matching a filter (for standalone/replica; not for `+srv` routers prior to certain versions).
- `--readPreference secondaryPreferred` – offload from primary if replica set.

---

## 4) Restore with `mongorestore`
> By default, restore **inserts** documents. To make the target match the dump *exactly*, use `--drop` (drops each collection before inserting).

### A) Restore everything from a folder
```bash
mongorestore \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  /path/to/backup/atlas-dump-YYYY-MM-DD_HH-MM
```

### B) Restore and replace existing data (safe exact restore)
```bash
mongorestore \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  --drop \
  /path/to/backup/atlas-dump-YYYY-MM-DD_HH-MM
```

### C) Restore a single database to the same name
```bash
mongorestore \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  --db myDatabase \
  /path/to/backup/atlas-dump-YYYY-MM-DD_HH-MM/myDatabase
```

### D) Restore a DB under a **new name** (rename on restore)
```bash
mongorestore \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  --nsFrom="myDatabase.*" \
  --nsTo="myDatabaseRestored.*" \
  /path/to/backup/atlas-dump-YYYY-MM-DD_HH-MM
```

### E) Restore a single collection
```bash
mongorestore \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  --db myDatabase \
  --collection myCollection \
  /path/to/backup/atlas-dump-YYYY-MM-DD_HH-MM/myDatabase/myCollection.bson
```

### F) Restore from a compressed archive
```bash
gunzip -c myDatabase-YYYY-MM-DD_HH-MM.archive.gz | \
  mongorestore --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" --archive --drop
```

### G) Useful options
- `--nsInclude 'db.coll*'` – include only matching namespaces.
- `--nsExclude 'db.temp*'` – exclude namespaces.
- `--maintainInsertionOrder` – predictable insert order for debugging.

---

## 5) Verification Checklist
- **Counts**: compare document counts before/after
  ```bash
  mongosh "mongodb+srv://<USER>:<PASS>@cluster0.../myDatabase" --eval 'db.myCollection.countDocuments()'
  ```
- **Indexes**: confirm indexes exist after restore (created from `.metadata.json`).
- **Sample queries**: run a few critical business queries.

---

## 6) Automation (Daily Backups)
Create `/usr/local/bin/atlas_backup.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
STAMP=$(date +%F_%H-%M)
OUT=/var/backups/mongo/atlas-$STAMP
mkdir -p "$OUT"

mongodump \
  --uri="mongodb+srv://<USER>:<PASS>@cluster0.xxxxx.mongodb.net/" \
  --out="$OUT"

tar -czf "$OUT.tar.gz" -C "$(dirname "$OUT")" "$(basename "$OUT")"
rm -rf "$OUT"
echo "Backup written: $OUT.tar.gz"
```

Cron (daily 02:00):
```bash
sudo crontab -e
# Add:
0 2 * * * /usr/local/bin/atlas_backup.sh >> /var/log/atlas_backup.log 2>&1
```

---

## 7) Security Best Practices
- Never commit connection strings with passwords; use environment variables or a `.env` file with correct permissions.
- Restrict Atlas user to least privilege (readAnyDatabase for dump; dbOwner on target DB for restore).
- Lock down IP access to trusted sources only.
- Encrypt archives at rest (e.g., `gpg`, S3 SSE if uploading to S3).

---

## 8) Common Errors & Fixes
- **`(Unauthorized)`** → credentials lack required roles.
- **`connection <timeout>`** → IP not whitelisted in Atlas, or VPC egress blocked.
- **`duplicate key error` during restore** → run with `--drop` to replace existing data.
- **`Failed: error writing data`** → insufficient disk space; check `df -h`.

---

## 9) Quick Reference
- **Full dump (all DBs)**
  ```bash
  mongodump --uri="mongodb+srv://<USER>:<PASS>@cluster0.../" --out=/backups/atlas-$(date +%F_%H-%M)
  ```
- **Exact restore (replace existing)**
  ```bash
  mongorestore --uri="mongodb+srv://<USER>:<PASS>@cluster0.../" --drop /backups/atlas-YYYY-MM-DD_HH-MM
  ```
- **Archive dump + restore**
  ```bash
  mongodump --uri="mongodb+srv://<USER>:<PASS>@cluster0.../myDB" --archive | gzip > myDB.archive.gz
  gunzip -c myDB.archive.gz | mongorestore --uri="mongodb+srv://<USER>:<PASS>@cluster0.../" --archive --drop
  ```

---

**Tip:** For large datasets, run backups from an EC2 in the same AWS region as Atlas to minimize latency and egress costs.


