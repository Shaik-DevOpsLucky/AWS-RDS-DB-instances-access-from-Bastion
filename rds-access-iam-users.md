# AWS RDS Read-Only Access Configuration for QA Team

## Objective

This document outlines the implementation steps to provide secure read-only access to the AWS RDS PostgreSQL database for QA users using:

* AWS IAM Identity Center
* AWS Secrets Manager
* Amazon RDS Query Editor
* PostgreSQL Read-Only User Permissions

This setup ensures QA users can:

✅ View and query database tables
✅ Execute SELECT statements
✅ Access RDS Query Editor securely

This setup prevents QA users from:

❌ Creating/modifying/deleting database objects
❌ Using admin credentials
❌ Performing write operations on the database

---

# Architecture Overview

```text
QA User
   ↓
IAM Identity Center
   ↓
Permission Set
   ↓
RDS Query Editor
   ↓
Secrets Manager Secret
   ↓
PostgreSQL Read-Only User
```

---

# Prerequisites

* AWS IAM Identity Center enabled
* AWS RDS PostgreSQL instance/cluster available
* AWS Secrets Manager enabled
* RDS Query Editor enabled
* PostgreSQL admin access available

---

# Step 1 — Create PostgreSQL Read-Only User

Login to PostgreSQL using admin credentials:

```bash
psql "host=<RDS-ENDPOINT> \
port=5432 \
dbname=app1_main \
user=app1_admin \
sslrootcert=/certs/global-bundle.pem \
sslmode=verify-full"
```

Create read-only user:

```sql
CREATE USER app1_qa WITH PASSWORD 'StrongPassword123';
```

Grant database connection access:

```sql
GRANT CONNECT ON DATABASE app1_main TO app1_qa;
```

Grant schema usage:

```sql
GRANT USAGE ON SCHEMA public TO app1_qa;
```

Grant read-only access to all existing tables:

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app1_qa;
```

Grant read-only access to future tables:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO app1_qa;
```

---

# Step 2 — Validate Read-Only Permissions

Login using QA user:

```bash
psql "host=<RDS-ENDPOINT> \
port=5432 \
dbname=app1_main \
user=app1_qa \
sslrootcert=/certs/global-bundle.pem \
sslmode=verify-full"
```

Validate read access:

```sql
SELECT * FROM cache_rec LIMIT 5;
```

Validate write restriction:

```sql
CREATE TABLE test(id int);
DELETE FROM cache_rec;
UPDATE cache_rec SET id = 1;
INSERT INTO cache_rec VALUES (...);
```

Expected result:

```text
ERROR: permission denied
```

---

# Step 3 — Store Credentials in AWS Secrets Manager

Navigate to:

```text
AWS Console
→ Secrets Manager
→ Store new secret
```

Choose:

```text
Credentials for Amazon RDS database
```

Provide:

```text
Username: app1_qa
Password: <PASSWORD>
```

Associate secret with RDS instance:

```text
app1-rds-dev
```

Secret name:

```text
qa-rds-secret
```

Save the secret.

---

# Step 4 — Create Custom IAM Policy

Navigate to:

```text
IAM
→ Policies
→ Create Policy
→ JSON
```

Use the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyQASecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-2:<ACCOUNT-ID>:secret:qa-rds-secret*"
    },
    {
      "Sid": "AllowRDSQueryEditor",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "rds-data:ExecuteStatement",
        "rds-data:BatchExecuteStatement",
        "rds-db:connect"
      ],
      "Resource": "*"
    }
  ]
}
```

Replace:

```text
<ACCOUNT-ID>
```

with the AWS Account ID.

Policy Name:

```text
QA-RDS-ReadOnly
```

---

# Step 5 — Create IAM Identity Center Group

Navigate to:

```text
IAM Identity Center
→ Groups
→ Create Group
```

Group name:

```text
QA-Team
```

Add QA users to this group.

---

# Step 6 — Create Permission Set

Navigate to:

```text
IAM Identity Center
→ Permission Sets
→ Create Permission Set
```

Choose:

```text
Custom Permission Set
```

Permission set name:

```text
QA-RDS-ReadOnly
```

Attach custom policy:

```text
QA-RDS-ReadOnly
```

Save the permission set.

---

# Step 7 — Assign Permission Set to QA Group

Navigate to:

```text
IAM Identity Center
→ AWS Accounts
→ Select AWS Account
→ Assign Users or Groups
```

Assign:

```text
Group: QA-Team
Permission Set: QA-RDS-ReadOnly
```

Complete assignment.

---

# Step 8 — QA User Login Process

QA users log in via:

```text
AWS IAM Identity Center Portal
```

Navigate to:

```text
RDS → Query Editor
```

Use:

```text
Database Instance : app1-rds-dev
Database Name     : app1_main
Username          : app1_qa
Secret            : qa-rds-secret
```

---

# Security Controls Implemented

| Control                        | Status |
| ------------------------------ | ------ |
| Separate DB user for QA        | ✅      |
| Read-only access only          | ✅      |
| No admin credential sharing    | ✅      |
| Secret restricted via IAM      | ✅      |
| Query Editor access controlled | ✅      |
| SSL-enabled DB connectivity    | ✅      |
| MFA recommended                | ✅      |

---

# Important Recommendations

## Do NOT assign:

```text
AdministratorAccess
AmazonRDSFullAccess
SecretsManagerReadWrite
```

to QA users.

---

# Recommended SSL Connection

Use SSL verification while connecting to PostgreSQL:

```bash
psql 'host=<RDS-ENDPOINT> \
port=5432 \
user=app1_qa \
dbname=app1_main \
sslrootcert=/certs/global-bundle.pem \
sslmode=verify-full'
```

---

# Final Outcome

QA users can:

✅ Execute SELECT queries
✅ View required database data
✅ Access RDS Query Editor securely

QA users cannot:

❌ Modify database records
❌ Delete tables/data
❌ Create objects
❌ Access admin credentials
❌ Use `app1_admin` account

---

# Future Enhancements (Optional)

* Restrict Query Editor access to specific RDS resource ARNs
* Enable IAM DB Authentication
* Enable CloudTrail monitoring for DB access auditing
* Implement automatic password rotation via Secrets Manager
* Configure session timeout and MFA enforcement policies

# Prepared by:
*Shaik Moulali*
# DevOps Lead
