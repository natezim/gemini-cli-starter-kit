# Resource Monitoring Tool (RMT)

## What RMT is
Gathers Tableau Server monitoring data. Requires Advanced Management license.
Identifies causes of slow load times, extract failures, critical issues.

Two components:
- Agent on each Tableau Server node
- RMT Server that processes + hosts web service

Can monitor multiple clusters from one RMT instance (separate environments).

## Architecture
- RabbitMQ message queue collects agent data → RMT Server → RMT PostgreSQL repository
- Since 2022.3: can use external PostgreSQL (AWS RDS) + external RabbitMQ (AWS AMQ)
- RMT accesses Tableau Server Repository directly "for performance reasons"
  → requires `readonly` password; treat RMT as additional consumer of repository access

## RMT repository schema
Internal table names NOT publicly documented. Access via official `.tds` data sources:

| .tds Surface | What it covers |
|---|---|
| Background Tasks | Extract refreshes, subscriptions, flows |
| Data Queries | Queries executed by Tableau Server |
| Gateway Requests | HTTP/VizQL session request data |
| Incidents | Issues detected by RMT |
| Server Performance | Hardware and process metrics (CPU, memory, disk) |
| Tableau Entities | Sites, projects, workbooks, views collected by RMT |

## Enabling read-only access
```bash
rmtadmin data-access ReadOnly
# Restart RMT DB
# Retrieve password: db.readOnlyPassword
```

For 2022.2 and earlier: edit `postgresql.conf` + `pg_hba.conf` (SCRAM-SHA-256), restart DB.
Security-sensitive — opening RMT DB access requires admin action.

## When to use RMT vs Repository

**Use Tableau Server Repository for:**
- Content inventory, ownership, permissions
- Auditing (historical_events)
- Scheduling (tasks, schedules, background_jobs)

**Use RMT for:**
- Hardware/process health (CPU, memory, disk)
- Incident tracking
- Workload bottlenecks
- Request/query performance (Gateway Requests, Data Queries)

RMT's separate database reduces contention on Tableau Server's operational repository.
