# Import CSV Data into PostgreSQL `trim` Table via Bastion Host

## Prerequisites

* Bastion server access (`.pem` key)
* PostgreSQL client installed on bastion
* CSV file available locally
* Access to RDS database from bastion
* PostgreSQL credentials

---

# Step 1: Navigate to CSV Location

```bash
cd ~/Downloads
```

---

# Step 2: Set PEM File Permissions

```bash
chmod 400 bastion-key.pem
```

---

# Step 3: Upload CSV File to Bastion Server

```bash
scp -i bastion-key.pem cahe_rec1.csv ubuntu@public-ip:/home/ubuntu/cahe_rec1.csv
```

---

# Step 4: Connect to Bastion Server

```bash
ssh -i sewb-prod-bastion-key.pem ubuntu@public-ip
```

---

# Step 5: Verify CSV File

```bash
ls -lh /home/ubuntu/cahe_rec1.csv
```

---

# Step 6: Install PostgreSQL Client (if not installed)

```bash
sudo apt update
sudo apt install postgresql-client -y
```

---

# Step 7: Connect to PostgreSQL Database

```bash
psql -h rds-prod-instance-1.jfdjoorur.ap-southeast-2.rds.amazonaws.com -U admin -d main
```

---

# Step 8: Verify Table Structure

```sql
\d trim
```

---

# Step 9: Take Backup of Existing Table

```bash
pg_dump -h sewb-rds-prod-instance-1.jfdjoorur.ap-southeast-2.rds.amazonaws.com -U admin -d main -t trim > trim_backup_$(date +%F_%H%M).sql
```

---

# Step 10: Check Existing Record Count

```sql
SELECT COUNT(*) FROM trim;
```

Example existing count:

```text
56
```

---

# Step 11: Create Temporary Staging Table

```sql
CREATE TEMP TABLE trim_stage
(LIKE trim INCLUDING DEFAULTS);
```

---

# Step 12: Import CSV into Staging Table

```sql
\copy trim_stage FROM '/home/ubuntu/cahe_rec1.csv' WITH (FORMAT csv, HEADER true, QUOTE '"', ESCAPE '"');
```

Expected output:

```text
COPY 76
```

---

# Step 13: Verify Imported CSV Row Count

```sql
SELECT COUNT(*) FROM trim_stage;
```

Expected:

```text
76
```

---

# Step 14: Check Duplicate Records

```sql
SELECT COUNT(*) AS duplicate_count
FROM trim_stage s
JOIN trim c
ON c.hash_key = s.hash_key;
```

Expected:

```text
5
```

---

# Step 15: Insert Only New Records into Actual Table

```sql
INSERT INTO trim
SELECT *
FROM trim_stage
ON CONFLICT (hash_key) DO NOTHING;
```

Expected:

```text
INSERT 0 71
```

---

# Step 16: Verify Final Record Count

```sql
SELECT COUNT(*) FROM trim;
```

Expected:

```text
127
```

---

# Step 17: Remove Temporary Table (Optional)

```sql
DROP TABLE IF EXISTS trim_stage;
```

Note:

* `trim_stage` is a TEMP table.
* PostgreSQL automatically deletes it after session ends.

---

# Step 18: Exit PostgreSQL

```sql
\q
```

---

# Summary

| Item                   | Count |
| ---------------------- | ----- |
| Existing Records       | 56    |
| CSV Records            | 76    |
| Duplicate Records      | 5     |
| Newly Inserted Records | 71    |
| Final Total Records    | 127   |

---

# Important Notes

* Always take DB backup before import.
* Perform testing in Dev/Staging before Production.
* `hash_key` is primary key in `trim`.
* Duplicate records are skipped safely using:

```sql
ON CONFLICT (hash_key) DO NOTHING;
```

* CSV contains JSON fields such as:

  * `diet_option`
  * `exe_option`
  * `product_option`
  * `diet_video`

* Import performed through bastion because RDS is deployed in private subnet.

# Prepared by:
*Shaik Moulali*
# DevOps Consultant

