# MongoDB Backup & Restore – Quick Cheatsheet

## 🔹 Backup (Dump)
- **All DBs**
```bash
mongodump --uri="mongodb+srv://<user>:<pass>@cluster0.xxxxx.mongodb.net/" --out=/path/to/backup
```

- **Single DB**
```bash
mongodump --uri="mongodb+srv://<user>:<pass>@cluster0.xxxxx.mongodb.net/myDB" --out=/path/to/backup
```

## 🔹 Restore
- **Basic restore**
```bash
mongorestore --uri="mongodb+srv://<user>:<pass>@cluster0.xxxxx.mongodb.net" /path/to/backup
```

- **Exact restore (replace existing)**
```bash
mongorestore --uri="mongodb+srv://<user>:<pass>@cluster0.xxxxx.mongodb.net" --drop /path/to/backup
```

## 🔹 Single Collection
- **Dump**
```bash
mongodump --uri="..." --db myDB --collection myColl --out=/path/to/backup
```

- **Restore**
```bash
mongorestore --uri="..." --db myDB --collection myColl /path/to/backup/myDB/myColl.bson --drop
```

## 🔹 Compressed Archive (one file)
- **Dump (gzip archive)**
```bash
mongodump --uri=".../myDB" --archive | gzip > myDB-$(date +%F_%H-%M).archive.gz
```

- **Restore from archive**
```bash
gunzip -c myDB-YYYY-MM-DD_HH-MM.archive.gz | mongorestore --uri="..." --archive --drop
```

## 🔹 Tips
- Whitelist your IP in Atlas before connecting.  
- Use `--drop` carefully (erases existing collections before restore).  
- For large datasets, run from an EC2 in the same region as Atlas.  
