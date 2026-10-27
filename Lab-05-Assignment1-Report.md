# Lab 5 — Assignment 1 Report: RDS Migration for the Lab 4 Matomo App

**Course**: Cloud Computing, T.E. Computer, Sem V (2026-2027) | **Account**: calista-labs (386347568553) | **Region**: ap-south-1
**EC2**: matomo-lab4 (i-077933894a1f47c75), http://15.252.100.78/ | **RDS**: matomo-rds | **Date**: 2026-08-22

## Architecture

Browser -> HTTP:80 -> EC2 (matomo-lab4: Nginx + PHP-FPM 8.5 + Matomo) -> TLS, MySQL protocol, port 3306 -> RDS matomo-rds (MariaDB 11.8.8). Security group `matomo-rds-sg` restricts inbound 3306 to the EC2 instance's security group only — no 0.0.0.0/0.

Lab 4's app runs MariaDB (SQL) — Case A of the brief. Deployed engine is **MariaDB 11.8.8**, not generic MySQL, because Matomo's dump uses the MariaDB-only `utf8mb4_uca1400_ai_ci` collation (no MySQL equivalent). MariaDB is literally the same DB type as Lab 4 and is a first-class RDS engine; the restore completed with zero errors, confirming this was correct.

## Database & security settings

| Setting | Value |
|---|---|
| Engine / class | MariaDB 11.8.8, db.t3.micro (free-tier) |
| Deployment | Single-AZ, 20 GiB gp3, autoscaling off |
| Endpoint | matomo-rds.chu4e4kcinv9.ap-south-1.rds.amazonaws.com:3306 |
| Public access | No |
| Inbound rule | 3306 from EC2 security group (sg-06e7da1b08cdd44a4) only |
| App DB user | matomo_app — SELECT/INSERT/UPDATE/DELETE/CREATE/INDEX/DROP/ALTER/CREATE TEMP TABLES/LOCK TABLES on matomo.* only; no FILE, no GRANT OPTION |
| TLS | Enabled, RDS CA bundle installed on EC2 |

## Migration & validation

Fresh `mariadb-dump --single-transaction --routines --triggers` from EC2, restored directly into RDS. Row counts matched exactly before/after: `matomo_site` 1→1, `matomo_log_visit` 1→1. Config cutover (`config.ini.php` `[database]` section: host, TLS) confirmed working via HTTP 200 and the Matomo sign-in page rendering against RDS.

## CRUD evidence (all via the running EC2 app, Administration -> Websites -> Manage)

| Op | Action | Result |
|---|---|---|
| Create | Added website "Lab5-RDS-Test" | New row in matomo_site on RDS |
| Read | Viewed Manage Websites list | Site listed live from RDS |
| Update | Renamed to "Lab5-RDS-Test-Updated" | Change reflected immediately |
| Delete | Deleted the test site | Site count back to 1 |

Full screenshots for every step (instance running, RDS created, SG lockdown, all 4 CRUD ops) are in `Lab5-Screenshots.docx`.

## Requirement checklist

- [x] Deploy AWS RDS (MariaDB — justified deviation from "(MySQL/PostgreSQL)" wording, see above)
- [x] Schema with 2+ tables (Matomo has 40+)
- [x] Connect the Lab 4 EC2 app to RDS
- [x] Demonstrate all CRUD from the running EC2 app
- [x] RDS inbound restricted to the EC2 security group only, no 0.0.0.0/0
