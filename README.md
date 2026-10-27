# Lab 5 — Assignment 1: RDS Migration for the Lab 4 Matomo App

Deploys the same database type Lab 4 used (SQL) on AWS RDS and connects it to the existing
Lab 4 EC2 application (Matomo), per the brief's Case A requirements.

- **EC2 app URL**: http://15.252.100.78/
- **EC2 instance**: `matomo-lab4` (`i-077933894a1f47c75`)
- **RDS instance**: `matomo-rds`, engine **MariaDB 11.8.8**, `db.t3.micro`, single-AZ, 20 GiB gp3
- **RDS endpoint**: `matomo-rds.chu4e4kcinv9.ap-south-1.rds.amazonaws.com:3306`

## Why MariaDB, not "MySQL/PostgreSQL"

The brief's Case A lists "(MySQL/PostgreSQL)" as the RDS engine examples. Lab 4's app runs on
**MariaDB**, not MySQL — and Matomo's schema dump uses `utf8mb4_uca1400_ai_ci`, a MariaDB-only
collation with no MySQL equivalent (MySQL tops out at `utf8mb4_0900_ai_ci`). Deploying RDS
MySQL would have made every `CREATE TABLE` statement in the restore fail with "Unknown
collation." RDS offers MariaDB as a first-class engine option, so that's what was deployed —
this is literally "the same DB type as Lab 4," which is the requirement's actual intent, and
avoids a lossy collation rewrite. The restore completed with zero errors, confirming this was
the correct call.

## Architecture

```
Browser ──HTTP:80──> EC2 (matomo-lab4)
                        Nginx + PHP-FPM 8.5 + Matomo app
                              │
                              │ MySQL protocol, TLS, port 3306
                              ▼
                     RDS (matomo-rds) — MariaDB 11.8.8
                     Security group: matomo-rds-sg
                     Inbound 3306 restricted to EC2's security group only
                     (sg-06e7da1b08cdd44a4) — no 0.0.0.0/0
```

## Deployment steps

1. **Create the RDS instance** (AWS Console → RDS → Create database):
   - Engine: MariaDB 11.8.8, template Dev/Test (this account has a "Free Tier" plan cap —
     backup retention had to be set to 1 day, 7 days was rejected)
   - Instance class: `db.t3.micro` (Burstable, free-tier eligible), Single-AZ, no standby
   - Storage: 20 GiB General Purpose SSD (gp3), no autoscaling
   - Connectivity: same default VPC as the EC2 instance, **Public access: No**
   - New security group `matomo-rds-sg`, initial database name `matomo`

2. **Lock down the security group**: edited `matomo-rds-sg`'s inbound rule so the only
   allowed source for port 3306 is `matomo-lab4`'s own security group ID
   (`sg-06e7da1b08cdd44a4`) — not a CIDR block. Satisfies the brief's mandatory security rule.

3. **Create the least-privilege application user** on RDS (connected as the master user):
   ```sql
   CREATE USER 'matomo_app'@'%' IDENTIFIED BY '<app password>';
   GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER,
         CREATE TEMPORARY TABLES, LOCK TABLES ON matomo.* TO 'matomo_app'@'%';
   FLUSH PRIVILEGES;
   ```
   Same grant set the local MariaDB user had in Lab 4 — no `FILE`, no `GRANT OPTION`.

4. **Migrate the data** — fresh logical dump from the EC2 box, restored straight into RDS:
   ```bash
   sudo sh -c "mariadb-dump --single-transaction --routines --triggers matomo | gzip > /var/backups/matomo/matomo-final.sql.gz"
   gunzip -c /var/backups/matomo/matomo-final.sql.gz | mysql -h matomo-rds.chu4e4kcinv9.ap-south-1.rds.amazonaws.com -u admin -p matomo
   ```
   Verified row counts matched exactly between local and RDS: `matomo_site` = 1,
   `matomo_log_visit` = 1 on both sides.

5. **Enable TLS**: downloaded the RDS CA bundle to the EC2 box
   (`/etc/ssl/certs/aws-rds-global-bundle.pem`) and added `enable_ssl = 1` /
   `ssl_ca = "..."` to Matomo's database config.

6. **Cut Matomo over to RDS**: updated `/var/www/matomo/config/config.ini.php`'s
   `[database]` section (see [`config.ini.php.redacted`](./config.ini.php.redacted) for the
   full, secret-redacted section). Old local config backed up as
   `config.ini.php.bak-pre-rds` on the instance for rollback.

## CRUD proof

Demonstrated all four operations live through the running EC2 app (Administration → Websites
→ Manage), each verified against RDS:

| Operation | Action taken | Result |
|---|---|---|
| Create | Added website "Lab5-RDS-Test" | New row in `matomo_site` (RDS), site count 1→2 |
| Read | Viewed Manage Websites list | Site listed live from RDS |
| Update | Renamed to "Lab5-RDS-Test-Updated" | `UPDATE` reflected immediately |
| Delete | Deleted the test site | Site count back to 1 |

Full screenshot evidence (EC2 app running, RDS instance deployed, security group lockdown,
and all four CRUD steps) is in `Lab5-Screenshots.docx` in the parent `lab5/` folder — see the
1-2 page PDF report (`Lab-05-Assignment1-Report.pdf`) in this repo for the condensed writeup.

## Security summary

- RDS **not publicly accessible** (`Public access: No`)
- Inbound 3306 restricted to the EC2 security group only, no `0.0.0.0/0`
- Application DB user has least-privilege grants only (no `FILE`, no `GRANT OPTION`)
- TLS enabled for the app↔RDS connection using the AWS RDS CA bundle
- No secrets committed to this repo — passwords are redacted in
  `config.ini.php.redacted`; the EC2 SSH key is intentionally excluded entirely
