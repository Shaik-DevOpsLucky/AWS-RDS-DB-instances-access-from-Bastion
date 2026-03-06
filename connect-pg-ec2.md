To connect **AWS RDS PostgreSQL** from a **Bastion (Jump) Server**, you need to ensure **network access, security groups, and PostgreSQL client** are configured.

Assume architecture:

```
Local Machine
      │
      │ SSH
      ▼
Bastion Host (EC2 - Public Subnet)
      │
      │ Private Network
      ▼
RDS PostgreSQL (Private Subnet)
```

This setup is common in **secure AWS architectures** where the database is not publicly accessible.

---

# 1️⃣ Verify Security Groups

First check the **RDS Security Group**.

Allow PostgreSQL access **from Bastion security group**.

Example rule:

| Type       | Protocol | Port | Source     |
| ---------- | -------- | ---- | ---------- |
| PostgreSQL | TCP      | 5432 | Bastion-SG |

So your **Amazon RDS PostgreSQL instance** only accepts traffic from the **Amazon EC2 Bastion host**.

---

# 2️⃣ Install PostgreSQL Client on Bastion

Login to Bastion:

```bash
ssh -i bastion.pem ec2-user@<Bastion-Public-IP>
```

Install PostgreSQL client.

### Amazon Linux

```bash
sudo yum install postgresql15 -y
```

### Ubuntu

```bash
sudo apt update
sudo apt install postgresql-client -y
sudo systemctl status postgresql
sudo -i -u postgres
psql
ALTER USER postgres WITH PASSWORD 'new_password';
```
# Reference:https://medium.com/@rhetal/basic-postgresql-setup-installation-users-access-and-backup-on-ubuntu-24-04-8c328dca9ac0

---

# 3️⃣ Get RDS Endpoint

Go to **Amazon RDS → Databases → Endpoint**

Example:

```
shaik-postgres.c1abcde.us-east-1.rds.amazonaws.com
```

---

# 4️⃣ Connect to Database

Run this command from Bastion:

```bash
psql -h shaik-postgres.c1abcde.us-east-1.rds.amazonaws.com \
-U postgres \
-d postgres \
-p 5432
```

Then enter password.

---

# 5️⃣ Test Connection

Once connected you will see:

```
postgres=>
```

Run:

```sql
\l
```

to list databases.

---

# 6️⃣ Troubleshooting

### ❌ Connection Timeout

Check:

* RDS security group
* Bastion security group
* VPC routing

### ❌ Could not translate hostname

Check **DNS resolution inside VPC**.

### ❌ password authentication failed

Check:

* username
* password
* database name

---

# 7️⃣ Best Practice (Production)

Instead of opening DB access, use **SSH Tunnel**:

```bash
ssh -i bastion.pem -L 5432:<RDS-ENDPOINT>:5432 ec2-user@<BASTION-IP>
```

Then connect locally:

```bash
psql -h localhost -U postgres
```

---

✅ **Production architecture usually looks like this:**

```
Internet
   │
   ▼
Bastion Host (Public Subnet)
   │
   ▼
Application Servers (Private Subnet)
   │
   ▼
RDS PostgreSQL (Private Subnet)
```

---

# Prepared by:
*Shaik Moulali*
