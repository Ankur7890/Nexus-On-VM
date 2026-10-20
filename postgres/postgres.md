## 1. PostgreSQL setup

Run the following on the PostgreSQL VM.

### Install PostgreSQL 18

```bash
sudo dnf update -y
sudo dnf module list postgresql
sudo dnf module install -y postgresql:18/server
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
sudo systemctl status postgresql
```

### Configure network access

Edit `/var/lib/pgsql/data/postgresql.conf` and configure PostgreSQL to listen on the required network interface. For my setup:

```conf
listen_addresses = '*'
```
[NOTE]
This allows PostgreSQL to accept connections on available interfaces. Access must still be restricted using firewall rules, security groups, and PostgreSQL authentication.

Edit `/var/lib/pgsql/data/pg_hba.conf`, replace the network with the range that contains only the intended Nexus clients:

```conf
host    nexus    nexus    10.0.0.0/16    scram-sha-256
```

[NOTE]
Here, 10.0.0.0/16 is nexus machine range, where nexus nexus are DB user and Database respectively. This is very broad range should be properly restricted in actual workload.
This rule allows the nexus user to connect to the nexus database from the specified network using SCRAM password authentication. 

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
sudo ss -lntp | grep 5432
```

### Create the database and user

Use a strong password and do not commit it to Git:

```bash
sudo -u postgres psql
```

At the `psql` prompt:

```sql
CREATE USER nexus WITH PASSWORD '<REPLACE_WITH_STRONG_PASSWORD>';
CREATE DATABASE nexus OWNER nexus;
\c nexus
```

Generally `pg_trgm` was initially unavailable, install the PostgreSQL contrib package. On the RHEL module stream used, the package was:

```bash
sudo dnf install -y postgresql-contrib
```

Then connect to the `nexus` database and retry:

```sql
CREATE SCHEMA nexus AUTHORIZATION nexus;
CREATE EXTENSION IF NOT EXISTS pg_trgm SCHEMA nexus;
```

Verify the extension:

```sql
\dx
####Expected Outputs####
nexus=> SELECT extname,
       extnamespace::regnamespace AS schema,
       extowner::regrole AS owner
FROM pg_extension
WHERE extname = 'pg_trgm';
 extname | schema | owner 
---------+--------+-------
 pg_trgm | nexus  | nexus
(1 row)
nexus=> SELECT current_database(), current_user;
 current_database | current_user 
------------------+--------------
 nexus            | nexus
(1 row)
nexus=> \q

```

Exit `psql` with `\q`.

### Validate connectivity from a Nexus VM

Install `nmap-ncat` if `nc` is not available, then test the database port:

```bash
sudo dnf install -y nmap-ncat
nc -zv <POSTGRESQL_PRIVATE_IP> 5432
```

This checks TCP connectivity only; it does not validate the database username, password, or schema.