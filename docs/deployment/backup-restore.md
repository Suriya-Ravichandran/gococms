# Backup & Restore

> Per-data-store backup and disaster-recovery strategy for GOCO CMS — MongoDB point-in-time recovery, Redis durability tiers, object-storage versioning/replication, encrypted off-site copies, scheduled jobs with distributed locks, and tested restore runbooks with defined RTO/RPO.

Stability: `stable` (procedures) · `beta` (per-tenant export/import tooling)

GOCO CMS is a stateful system: authoritative content lives in **MongoDB**, durable jobs and rate-limit/session state live in **Redis**, and binary media lives in **object storage** (Local / MinIO / Amazon S3). A credible backup plan treats each store on its own terms — you cannot `mongodump` your Redis queue, and an S3 versioning policy will not save a dropped MongoDB collection. This document defines what to back up, how, how often, how to encrypt it, and — most importantly — how to *restore* under pressure.

> **Warning**
> A backup you have never restored is a hypothesis, not a backup. Every procedure here ends in a **verification** step, and Section [Testing & Verifying Backups](#testing--verifying-backups) is mandatory, not optional.

---

## Recovery Objectives (RTO / RPO)

Two numbers drive every decision below.

- **RPO — Recovery Point Objective**: the maximum acceptable *data loss*, measured in time. "RPO = 5 min" means a disaster may cost you up to five minutes of writes.
- **RTO — Recovery Time Objective**: the maximum acceptable *downtime* to restore service.

GOCO's default targets per tier (override in `config/backup.php` and your SLA):

| Tier | Store | RPO | RTO | Mechanism |
|------|-------|-----|-----|-----------|
| Critical | MongoDB (content, users, billing) | ≤ 5 min | ≤ 30 min | Replica set + oplog PITR + hourly dumps |
| Critical | Object storage (media) | ≤ 1 min | ≤ 15 min | Versioning + cross-region replication |
| Durable | Redis (queue `jobs`, notifications) | ≤ 60 s | ≤ 10 min | AOF `everysec` + RDB snapshots |
| Ephemeral | Redis (cache, sessions, locks, rate-limit) | ∞ (rebuildable) | 0 (regenerates) | No backup — designed to be lost |
| Config | Secrets, `.env`, Traefik ACME | ≤ 1 day | ≤ 5 min | Encrypted vault + git-tracked non-secrets |

> **Note**
> RPO and RTO are business decisions, not engineering ones. Confirm these numbers with stakeholders, then engineer to meet them. Everything below is tuned to the defaults in this table.

```mermaid
flowchart LR
    subgraph Sources["Live Stores"]
        M[(MongoDB<br/>replica set)]
        R[(Redis<br/>durable + ephemeral)]
        O[(Object Storage<br/>MinIO / S3)]
        C[Config & Secrets]
    end
    subgraph Jobs["Backup Workers"]
        BW[OpenSwoole task worker<br/>+ Redis distributed lock]
        CR[cron / systemd timer]
    end
    subgraph Vault["Encrypted Off-site"]
        A[(S3 backup bucket<br/>SSE-KMS + versioning)]
        G[(Cross-region replica)]
    end
    M -- mongodump + oplog --> BW
    R -- BGSAVE / AOF copy --> BW
    O -- replication rule --> G
    C -- age/gpg encrypt --> BW
    CR --> BW
    BW -- age-encrypted tarballs --> A
    A -- replication --> G
    A -- lifecycle: expire/glacier --> A
```

---

## Backup Inventory

Before automating anything, enumerate what must survive a total-loss event. The `goco backup:inventory` command prints this table for the running deployment.

| Asset | Store | Durable? | Backup method | Default schedule |
|-------|-------|----------|---------------|------------------|
| Content, users, RBAC, settings | MongoDB | Yes | `mongodump` + oplog tail | Hourly full, continuous oplog |
| `page_revisions`, `audit_logs` | MongoDB | Yes | Included in dump | Hourly |
| Durable job queue (`jobs`) | Redis | Yes | AOF + RDB | AOF everysec, RDB 15 min |
| Notifications, delayed jobs (ZSET) | Redis | Yes | AOF + RDB | Same |
| Cache, sessions, locks, rate-limit | Redis | **No** | None (rebuildable) | — |
| Media, uploads, generated derivatives | Object storage | Yes | Versioning + replication | Continuous |
| `.env`, secrets, JWT keys | Config | Yes | Encrypted vault export | Daily |
| Traefik `acme.json` (Let's Encrypt certs) | Config | Yes | Encrypted file copy | Daily (regenerable) |
| `docker-compose.yml`, Traefik dynamic config | Config | Yes | Git repository | On change |

> **Tip**
> If it is not in this inventory, it is not backed up. Add new stateful services to `config/backup.php` the same day you deploy them.

---

## MongoDB Backup

MongoDB holds the authoritative state. GOCO layers three mechanisms with increasing granularity: logical dumps (portable), oplog-based **point-in-time recovery** (fine-grained), and volume/replica snapshots (fast, whole-cluster).

### Replica set as a prerequisite

Point-in-time recovery and consistent snapshots require a **replica set** (even a single-node set for dev). The compose service `mongodb` runs with `--replSet rs0`. Initialize once:

```bash
docker compose exec mongodb mongosh --eval '
  rs.initiate({
    _id: "rs0",
    members: [{ _id: 0, host: "mongodb:27017" }]
  })
'
```

Production uses a 3-member set (primary + 2 secondaries, or primary + secondary + arbiter) so that snapshots run against a secondary without loading the primary, and so failover itself is a recovery mechanism.

### Logical dumps — `mongodump` / `mongorestore`

Logical dumps are portable across versions and storage engines and are the backbone of the hourly backup. Always include `--oplog` on the dump so the snapshot is consistent to a single point even while writes continue.

```bash
# Consistent logical dump of the whole deployment DB, gzipped, with oplog.
docker compose exec -T mongodb mongodump \
  --uri="mongodb://mongodb:27017/?replicaSet=rs0" \
  --db=goco \
  --oplog \
  --gzip \
  --numParallelCollections=4 \
  --archive=/backup/gococms-$(date -u +%Y%m%dT%H%M%SZ).archive.gz
```

`--archive` writes a single stream (ideal for piping into encryption, see below). `--oplog` captures oplog entries during the dump so `mongorestore --oplogReplay` yields a snapshot consistent to the dump's end time.

Restore a logical dump:

```bash
# Full restore into a clean database. --drop replaces existing collections.
docker compose exec -T mongodb mongorestore \
  --uri="mongodb://mongodb:27017/?replicaSet=rs0" \
  --nsInclude="goco.*" \
  --drop \
  --gzip \
  --oplogReplay \
  --numParallelCollections=4 \
  --archive=/backup/gococms-20260718T030000Z.archive.gz
```

Restore a **single collection** (e.g. a designer dropped `widgets`):

```bash
docker compose exec -T mongodb mongorestore \
  --uri="mongodb://mongodb:27017/?replicaSet=rs0" \
  --nsInclude="goco.widgets" \
  --drop \
  --gzip \
  --archive=/backup/gococms-20260718T030000Z.archive.gz
```

> **Warning**
> `--drop` deletes the target collection before restoring. Never run a full `--drop` restore against a live production database without confirming you are restoring the correct archive — restore into a scratch database first (`--nsFrom="goco.*" --nsTo="goco_verify.*"`) and diff.

### Point-in-time recovery (PITR) via the oplog

Hourly dumps give an RPO of one hour — not good enough for the Critical tier. To hit **RPO ≤ 5 minutes**, GOCO continuously tails the replica-set **oplog** (`local.oplog.rs`) between full dumps. Recovery = restore the most recent full dump, then replay oplog entries up to the exact timestamp just before the incident.

The oplog tailer ships as `scripts/oplog-tail.php` and runs as a supervised process. Conceptually:

```bash
# Capture oplog slices every 5 minutes, bounded by the last full-dump timestamp.
docker compose exec -T mongodb mongodump \
  --uri="mongodb://mongodb:27017/?replicaSet=rs0" \
  --db=local --collection=oplog.rs \
  --query='{"ts": {"$gte": {"$timestamp": {"t": 1752807600, "i": 1}}}}' \
  --gzip \
  --archive=/backup/oplog/oplog-$(date -u +%Y%m%dT%H%M%SZ).bson.gz
```

To recover to a precise moment (say, the second *before* an accidental mass-delete at `2026-07-18T14:32:07Z`):

```bash
# 1. Restore the last good full dump taken BEFORE the incident.
mongorestore --drop --gzip --oplogReplay \
  --archive=/backup/gococms-20260718T140000Z.archive.gz

# 2. Replay oplog only up to the target timestamp (epoch seconds).
mongorestore \
  --oplogReplay \
  --oplogLimit=1752849127:1 \
  --dir=/backup/oplog-extracted/
```

`--oplogLimit=<seconds>:<ordinal>` stops replay *before* the given timestamp — set it to the moment just prior to the destructive operation, which you can pinpoint from `audit_logs` (see [Data Model](../architecture/data-model.md)).

> **Note**
> The oplog is a capped collection. Size it so it holds at least your full-dump interval plus generous slack (e.g. 24–48 h of writes). Check headroom with `rs.printReplicationInfo()` — "oplog first event time" must always predate your oldest full dump you might restore from.

### Physical snapshots (fast whole-cluster recovery)

For large databases where a logical restore would blow the RTO, take **filesystem/volume snapshots** of a secondary's data volume. Because writes are journaled, a snapshot of the WiredTiger data directory is crash-consistent. On cloud volumes use provider snapshots; on-prem use LVM or ZFS.

```bash
# Flush + fsync-lock a secondary, snapshot its volume, then unlock.
docker compose exec -T mongodb mongosh --eval 'db.fsyncLock()'
# ... take the volume/LVM/ZFS snapshot of the mongodb data volume here ...
zfs snapshot tank/mongodb@$(date -u +%Y%m%dT%H%M%SZ)
docker compose exec -T mongodb mongosh --eval 'db.fsyncUnlock()'
```

Snapshots restore in minutes regardless of dataset size (best RTO) but are less portable than dumps. Use snapshots for the primary recovery path and logical dumps for long-term retention and cross-version portability.

### MongoDB backup decision matrix

| Need | Use |
|------|-----|
| Portable, long-retention, per-collection restore | `mongodump --oplog --gzip --archive` |
| RPO ≤ 5 min / restore to exact second | Full dump + continuous oplog capture |
| Fastest whole-cluster restore (best RTO) | Volume/replica snapshot of a secondary |
| Move data between environments | `mongodump` → `mongorestore --nsFrom/--nsTo` |

---

## Redis Backup — Durable vs Ephemeral

Redis in GOCO stores **two categories of data with opposite backup requirements**. Backing up the wrong keyspaces wastes I/O and, worse, restoring stale locks or sessions after a failover causes bugs. See [Caching, Queue & Realtime](../architecture/caching-and-queue.md) for the full keyspace map.

| Keyspace / DB | Contents | Durability | On restore |
|---------------|----------|------------|------------|
| `db0` — cache | HTTP fragments, query cache, computed views | **Ephemeral** | Let it rebuild; never restore |
| `db0` — sessions | Auth sessions (also recoverable via re-login) | **Ephemeral** | Do not restore; users re-authenticate |
| `db0` — locks, rate-limit | Distributed locks, token buckets | **Ephemeral** | Must NOT survive a restore (stale locks deadlock) |
| `db1` — `jobs` queue, delayed ZSETs | Durable background jobs | **Durable** | Restore required — lost jobs = lost work |
| `db1` — notifications outbox | Pending user notifications | **Durable** | Restore required |

> **Warning**
> Never restore an old RDB that contains lock keys (`lock:*`) or rate-limit buckets into a live cluster — a resurrected lock with a long TTL will block workers until it expires. Isolate durable data on a dedicated logical DB (`db1`) or, better, a **separate Redis instance** (`redis-durable`) so backups target only durable keys.

### Recommended topology

Run two Redis roles:

- `redis` — cache/sessions/locks/rate-limit. `save ""` (no RDB), `appendonly no`. Purely in-memory; loss is a non-event.
- `redis-durable` — queue + notifications. **AOF `everysec` + periodic RDB.** This is the only Redis you back up.

```conf
# redis-durable.conf — durability-first configuration
appendonly yes
appendfsync everysec          # ~1s worst-case data loss → meets RPO ≤ 60s
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
save 900 1                    # RDB snapshot as a coarse fallback
save 300 100
save 60 10000
dir /data
dbfilename dump.rdb
appenddirname appendonlydir
```

### RDB + AOF explained

- **RDB** is a point-in-time binary snapshot — compact, fast to load, but loses everything since the last save. Trigger one on demand with `BGSAVE`.
- **AOF** logs every write; with `appendfsync everysec` you lose at most ~1 second. AOF is the durable-tier backbone; RDB is a fast-loading fallback.

Take a consistent copy for off-site archival:

```bash
# Force a fresh RDB, then archive both RDB and AOF directory.
docker compose exec -T redis-durable redis-cli BGSAVE
# Wait until rdb_bgsave_in_progress:0
docker compose exec -T redis-durable sh -c \
  'while [ "$(redis-cli info persistence | grep -c rdb_bgsave_in_progress:1)" != "0" ]; do sleep 1; done'
# Copy the data volume contents (dump.rdb + appendonlydir/) into the backup stream.
docker compose exec -T redis-durable tar -czf - -C /data dump.rdb appendonlydir \
  > /backup/redis-durable-$(date -u +%Y%m%dT%H%M%SZ).tar.gz
```

Restore:

```bash
# Stop the durable Redis, replace its data dir, restart. Redis prefers AOF on boot.
docker compose stop redis-durable
docker run --rm -v gococms_redis_durable_data:/data -v /backup:/backup alpine \
  sh -c 'cd /data && rm -rf ./* && tar -xzf /backup/redis-durable-20260718T140000Z.tar.gz'
docker compose start redis-durable
docker compose exec redis-durable redis-cli DBSIZE   # sanity check
```

> **Note**
> When both AOF and RDB exist, Redis loads the **AOF** on startup (it is more complete). Ensure `appendonly yes` in the restored instance's config or it will load the older RDB instead.

---

## Object Storage Backup (MinIO / S3)

Media and uploads live behind GOCO's storage driver interface (Local, MinIO, Amazon S3 — see [Storage & Media](../architecture/storage.md)). Object storage is backed up **in place** through native features rather than dumps, because copying terabytes of media on a schedule is impractical.

### 1. Versioning — protection against overwrite/delete

Enable bucket versioning so every `PUT` and `DELETE` is recoverable. A "delete" writes a delete marker; the prior version remains.

```bash
# MinIO
mc version enable local/gococms-media

# Amazon S3
aws s3api put-bucket-versioning \
  --bucket gococms-media \
  --versioning-configuration Status=Enabled
```

Recover a specific overwritten object:

```bash
# List versions, then restore the desired version id.
aws s3api list-object-versions --bucket gococms-media --prefix workspaces/w_123/media/hero.jpg
aws s3api copy-object \
  --bucket gococms-media \
  --copy-source "gococms-media/workspaces/w_123/media/hero.jpg?versionId=3sL4kqtJ..." \
  --key workspaces/w_123/media/hero.jpg
```

### 2. Cross-region replication — protection against region loss

Replicate the media bucket to a second region/provider for DR. This is the object-storage equivalent of an off-site copy and is how you meet **RTO ≤ 15 min** for media (failover reads to the replica).

```bash
# MinIO active-active or one-way replication to a DR site.
mc replicate add local/gococms-media \
  --remote-bucket https://DR_KEY:DR_SECRET@dr.example.com/gococms-media-dr \
  --replicate "delete,delete-marker,existing-objects"
```

```json
// S3 replication rule (put-bucket-replication) — replicate everything, including deletes.
{
  "Role": "arn:aws:iam::123456789012:role/gococms-replication",
  "Rules": [{
    "ID": "media-dr",
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "DeleteMarkerReplication": { "Status": "Enabled" },
    "Destination": {
      "Bucket": "arn:aws:s3:::gococms-media-dr",
      "StorageClass": "STANDARD_IA"
    }
  }]
}
```

### 3. Lifecycle — cost control and retention

Expire noncurrent versions and transition cold media to cheaper storage so versioning does not grow unbounded.

```json
// S3 lifecycle configuration
{
  "Rules": [
    {
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 },
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 30, "StorageClass": "GLACIER" }
      ]
    },
    {
      "ID": "abort-incomplete-mpu",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }
  ]
}
```

### Media and derivatives

GOCO stores original uploads plus generated **derivatives** (thumbnails, responsive sizes, WebP/AVIF variants). Derivatives are **regenerable** from originals via `goco media:regenerate`, so you may exclude them from replication to cut cost — at the price of a CPU-bound rebuild after a restore. For fastest RTO, replicate everything; for lowest cost, replicate originals only and rebuild derivatives on demand.

> **Tip**
> Keep the **media metadata** (the `media` collection in MongoDB) and the **binary objects** consistent. A restore that rolls MongoDB back but not object storage — or vice versa — leaves dangling references. Run `goco media:reconcile` after any cross-store restore to detect and quarantine orphans.

---

## Config & Secrets Backup

The database is worthless without the keys to decrypt tokens and the config to boot the app.

Back up, separately from data and with stricter access control:

- `.env` / environment (see [Configuration](../getting-started/configuration.md)) — DB URIs, Redis URLs, `APP_KEY`, JWT signing keys, OAuth client secrets, S3 credentials.
- JWT/passkey signing keypairs (`config/keys/`).
- Traefik `acme.json` — Let's Encrypt certificates and account key (regenerable, but backing it up avoids rate-limit issues on mass re-issue).
- `docker-compose.yml` and Traefik dynamic config — track in **git**; these contain no secrets when secrets are injected via env/Docker secrets.

```bash
# Encrypted secrets bundle (never store this unencrypted).
tar -czf - .env config/keys traefik/acme.json \
  | age -r age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgs... \
  > /backup/secrets-$(date -u +%Y%m%dT%H%M%SZ).tar.gz.age
```

> **Warning**
> Secrets backups must live in a **separate vault** with a different access policy than data backups (ideally a dedicated KMS-protected bucket or a secrets manager like Vault/AWS Secrets Manager). A single compromised credential should never grant access to both encrypted data *and* the keys to decrypt it.

Non-secret config (compose files, Traefik static config, CI) belongs in the git repository — versioned, reviewed, and reproducible. See [Docker Architecture](./docker.md) and [Traefik Reverse Proxy](./traefik.md).

---

## Encryption of Backups

All backups leaving the trust boundary are **encrypted at rest and in transit**. GOCO standardizes on [`age`](https://github.com/FiloSottile/age) for portable, key-based encryption and relies on SSE-KMS for the archival bucket.

**Layered approach:**

1. **Client-side (`age`)** — encrypt the archive stream *before* it leaves the host, so plaintext never touches disk or network. Recipients are `age` public keys; only holders of the matching private key (stored in your secrets manager) can decrypt.
2. **Server-side (SSE-KMS)** — the backup bucket also enforces `aws:kms` encryption, giving key rotation, audit trails, and defense-in-depth.
3. **In transit** — TLS to the object store (enforced by bucket policy `aws:SecureTransport`).

Encrypt on the way out, decrypt on the way in:

```bash
# Encrypt a mongodump stream directly to the vault — no plaintext on disk.
mongodump --archive --gzip --oplog --db=goco \
  | age -r "$(cat /run/secrets/backup_age_pub)" \
  | aws s3 cp - s3://gococms-backups/mongo/gococms-$(date -u +%Y%m%dT%H%M%SZ).archive.gz.age \
      --sse aws:kms --sse-kms-key-id alias/gococms-backups

# Decrypt on restore.
aws s3 cp s3://gococms-backups/mongo/gococms-20260718T140000Z.archive.gz.age - \
  | age -d -i /run/secrets/backup_age_key \
  | mongorestore --archive --gzip --drop --oplogReplay
```

> **Note**
> Store the `age` **private key** where you can reach it *during a disaster* — not only inside the environment you are trying to recover. A key locked exclusively inside the down cluster is a classic unrecoverable-backup trap. Escrow it in a separate KMS/region.

---

## Scheduling Backups

Backups run on a schedule with **exactly-once** semantics across a multi-node cluster — two workers must never dump simultaneously. GOCO offers two schedulers; both acquire a **Redis distributed lock** before running.

### Option A — OpenSwoole task worker (recommended)

The runtime is already running under ZealPHP/OpenSwoole, so schedule backups inside the app using `App::onWorkerStart` + `App::tick`, guarded by `\ZealPHP\Store` / a Redis lock so only one worker in one node fires. Heavy work is dispatched to a **task worker** so the reactor is never blocked.

```php
<?php
// scripts/backup_scheduler.php — registered during app bootstrap.
use ZealPHP\App;
use ZealPHP\Store;

App::onWorkerStart(function ($server, $workerId) {
    // Only worker 0 arms the timer; the lock guards against multi-node races.
    if ($workerId !== 0) {
        return;
    }

    // Hourly MongoDB dump.
    App::tick(3600_000, function () use ($server) {
        // Cluster-wide distributed lock via Redis (SET NX PX). TTL > job duration.
        if (!\Goco\Queue\Lock::acquire('backup:mongo', ttlMs: 1_800_000)) {
            return; // another node already holds it
        }
        // Offload the blocking dump to a task worker.
        $server->task(['job' => 'backup.mongo', 'ts' => time()]);
    });

    // Continuous oplog capture for PITR (every 5 min).
    App::tick(300_000, function () use ($server) {
        if (\Goco\Queue\Lock::acquire('backup:oplog', ttlMs: 240_000)) {
            $server->task(['job' => 'backup.oplog', 'ts' => time()]);
        }
    });

    // Durable-Redis snapshot every 15 min.
    App::tick(900_000, function () use ($server) {
        if (\Goco\Queue\Lock::acquire('backup:redis', ttlMs: 600_000)) {
            $server->task(['job' => 'backup.redis', 'ts' => time()]);
        }
    });
});

// Task workers run the actual shell-outs off the reactor loop.
App::onTask(function ($server, $taskId, $srcWorkerId, $data) {
    match ($data['job']) {
        'backup.mongo' => \Goco\Backup\Runner::mongo(),
        'backup.oplog' => \Goco\Backup\Runner::oplog(),
        'backup.redis' => \Goco\Backup\Runner::redisDurable(),
        default => null,
    };
    // Emit an audit event + metric; failures raise an alert (see below).
    return ['job' => $data['job'], 'status' => 'ok'];
});
```

The `Hook` system emits `backup.started`, `backup.completed`, and `backup.failed`, so plugins (alerting, dashboards) can react — see [Event & Hook System](../architecture/event-hook-system.md).

### Option B — External cron / systemd timer

For operators who prefer OS-level scheduling, `goco backup:*` commands are cron-friendly and self-lock via the same Redis key.

```bash
# /etc/cron.d/gococms-backup
# m h dom mon dow user command
5  *    * * *  gococms  docker compose exec -T gococms goco backup:mongo   --lock
*/5 *   * * *  gococms  docker compose exec -T gococms goco backup:oplog   --lock
*/15 *  * * *  gococms  docker compose exec -T gococms goco backup:redis   --lock
0  2    * * *  gococms  docker compose exec -T gococms goco backup:secrets --lock
30 3    * * 0  gococms  docker compose exec -T gococms goco backup:verify         # weekly restore test
```

`--lock` acquires `backup:<type>` in Redis (`SET NX PX`) and exits quietly if another node holds it — so the same crontab can be deployed to every node without double-running.

> **Tip**
> Stagger schedules so MongoDB dumps, Redis snapshots, and lifecycle sweeps do not all fire on the same minute and saturate I/O. The defaults above intentionally offset by store.

### Retention (GFS)

Apply a Grandfather-Father-Son policy via object-storage lifecycle rules or `goco backup:prune`:

| Cadence | Keep |
|---------|------|
| Hourly (Mongo full) | 48 hours |
| Daily | 30 days |
| Weekly | 12 weeks |
| Monthly | 12 months |
| Yearly | 7 years (compliance) |

---

## Restore & Disaster-Recovery Runbooks

Runbooks are step-by-step, copy-pasteable, and rehearsed. Under pressure, follow the runbook — do not improvise.

### Runbook A — Accidental data deletion (single collection / tenant)

**Scenario:** an editor bulk-deleted pages, or a bad migration corrupted one collection. Cluster is healthy.

1. **Freeze writes** to the affected scope: put the website into maintenance mode (`goco site:maintenance on --website=<id>`), which returns 503 via Traefik.
2. **Pinpoint the moment** of damage from `audit_logs` — find the offending operation's timestamp.
3. **Restore into a scratch DB** (never straight over production):
   ```bash
   mongorestore --nsFrom="goco.pages" --nsTo="goco_recover.pages" \
     --gzip --archive=/backup/gococms-<pre-incident>.archive.gz
   ```
4. If sub-hour precision is needed, **replay oplog** into the scratch DB up to `--oplogLimit=<ts>`.
5. **Verify** the scratch data (counts, spot-check documents).
6. **Merge back**: copy the recovered documents into production (`goco restore:merge --from=goco_recover --collection=pages --scope=website:<id>`), or swap collections during a brief window.
7. `goco media:reconcile` if media references changed.
8. **Lift maintenance mode.** Record the incident.

**Target: RTO ≤ 30 min, RPO ≤ 5 min.**

### Runbook B — Full MongoDB loss

**Scenario:** the entire replica set is gone (volume corruption, deleted cluster).

1. Provision a fresh replica set (`docker compose up -d mongodb`, `rs.initiate`).
2. Choose the fastest viable source: **volume snapshot** (best RTO) if same-version/same-platform; otherwise the latest **logical archive**.
   ```bash
   aws s3 cp s3://gococms-backups/mongo/<latest>.archive.gz.age - \
     | age -d -i /run/secrets/backup_age_key \
     | mongorestore --archive --gzip --drop --oplogReplay
   ```
3. **Replay oplog slices** captured after that dump to reach the last-known-good second.
4. **Rebuild indexes** if not implied by the restore (`goco db:index:sync`).
5. **Point the app** at the restored set; Redis cache is cold — it repopulates on traffic (this is by design; do NOT restore cache).
6. **Re-verify** durable Redis (`jobs`) is intact or restore it (Runbook C).
7. Run `goco media:reconcile`. Bring traffic back gradually.

**Target: RTO ≤ 30 min, RPO ≤ 5 min.**

### Runbook C — Redis durable loss (lost jobs)

1. Stop `redis-durable`, restore its AOF/RDB tarball (see [Redis Backup](#redis-backup--durable-vs-ephemeral)), restart.
2. Verify `DBSIZE` and inspect the `jobs` queue length.
3. Do **not** restore ephemeral Redis — start it empty; cache/sessions/locks regenerate.
4. Resume workers; idempotent jobs safely re-run.

### Runbook D — Region / site failure (full DR)

1. **Object storage**: fail reads over to the cross-region replica bucket (update the storage driver endpoint or DNS).
2. **MongoDB**: promote the DR-region secondary (if a cross-region member exists) or restore latest archive into a DR cluster.
3. **Redis durable**: restore from the latest off-site tarball.
4. **Config/secrets**: decrypt the secrets bundle with the escrowed `age` key.
5. **Traefik**: bring up the proxy; Let's Encrypt re-issues certs (or restore `acme.json`), DNS points at the DR ingress.
6. Smoke-test, then cut traffic over.

**Target: RTO ≤ 60 min for whole-site.**

---

## Per-Tenant Export & Import

GOCO is multi-tenant (`workspace_id` + `website_id` on tenant-scoped docs — see [Multi-Tenancy](../architecture/multi-tenancy.md)). Beyond cluster-wide backups, operators frequently need to export or move a **single tenant**: offboarding, cloning to staging, or migrating an enterprise workspace to a database-per-workspace deployment.

The `goco tenant:export` command produces a self-contained, encrypted bundle: a filtered MongoDB dump (all tenant-scoped documents), the tenant's media objects, and a manifest recording schema/version compatibility.

```bash
# Export one website (content + media) to an encrypted portable bundle.
docker compose exec -T gococms goco tenant:export \
  --workspace=w_123 \
  --website=s_456 \
  --include=media \
  --encrypt-to=$(cat backup_age.pub) \
  --out=/backup/tenants/s_456-$(date -u +%Y%m%dT%H%M%SZ).goco.age
```

Under the hood the export filters every tenant-scoped collection by both IDs:

```bash
mongodump --db=goco --gzip --archive \
  --query='{ "workspace_id": "w_123", "website_id": "s_456" }' \
  # ...repeated per tenant-scoped collection; global collections excluded...
```

Import into another deployment (optionally **re-keying** IDs to avoid collisions):

```bash
docker compose exec -T gococms goco tenant:import \
  --file=/backup/tenants/s_456-20260718T140000Z.goco.age \
  --decrypt-key=/run/secrets/backup_age_key \
  --target-workspace=w_789 \
  --remap-website-id \
  --with-media
```

> **Warning**
> A raw `mongorestore` of a tenant bundle into a shared multi-tenant DB can **collide** on `_id` or leak across tenants. Always use `goco tenant:import`, which validates tenant scoping, optionally remaps IDs, transfers media, and runs `media:reconcile`. Direct restores are only safe for database-per-workspace targets.

Per-tenant point-in-time recovery combines a tenant export as the base with a **filtered oplog replay** (oplog entries matching the tenant's namespaces up to `--oplogLimit`), giving fine-grained recovery of one customer without touching neighbors.

---

## Testing & Verifying Backups

An untested backup is a liability. GOCO automates verification so failures surface *before* a real disaster.

### Automated weekly restore test

`goco backup:verify` (scheduled weekly, see cron above) performs a **full restore into a throwaway environment** and asserts integrity:

```bash
docker compose exec -T gococms goco backup:verify \
  --source=latest \
  --target=ephemeral \
  --checks="doc-counts,index-parity,checksum,smoke"
```

It performs:

1. **Decrypt + restore** the latest MongoDB archive into a disposable database.
2. **Doc-count parity** — compare per-collection counts against the source's recorded manifest (within tolerance for live drift).
3. **Index parity** — confirm every documented index exists post-restore (`db:index:sync --dry-run`).
4. **Checksum** — hash a sample of documents and compare to the manifest captured at dump time.
5. **Smoke test** — boot a throwaway app pointed at the restored DB and hit health/render endpoints.
6. **Redis** — load the durable tarball into a scratch instance, assert `jobs` queue integrity.
7. **Media** — reconcile a sample of `media` docs against object-storage HEAD requests.
8. Emit `backup.verify.passed` / `backup.verify.failed` hooks; a failure pages on-call.

### Backup health monitoring

Track and alert on:

- **Backup age** — alert if the newest MongoDB archive is older than 90 minutes (RPO breach risk).
- **Backup size delta** — a sudden size drop often signals a partial/corrupt dump.
- **Oplog window** — alert if the oplog's oldest entry approaches the newest full dump (PITR gap forming).
- **Replication lag** — cross-region object-storage replication lag beyond threshold.
- **Restore-test result** — the weekly `backup:verify` must pass; a miss is a Sev-2.

### DR game days

Schedule a quarterly **game day**: execute Runbook B or D against a staging clone from real backups, time it end-to-end, and confirm you meet the RTO/RPO in the [objectives table](#recovery-objectives-rto--rpo). Update the runbooks with anything that surprised you. A runbook is only trustworthy after a human has followed it under a stopwatch.

> **Tip**
> Record measured RTO/RPO from each game day in `docs/` alongside the incident log. Trends reveal creeping regressions (dataset growth pushing restore time past SLA) long before they bite in production.

---

## Related

- [Docker Architecture](./docker.md)
- [Traefik Reverse Proxy](./traefik.md)
- [Deployment Guide](./deployment-guide.md)
- [Scaling Strategy](./scaling.md)
- [MongoDB Data Layer](../architecture/database-mongodb.md)
- [Data Model (Collections & Indexes)](../architecture/data-model.md)
- [Caching, Queue & Realtime (Redis)](../architecture/caching-and-queue.md)
- [Storage & Media](../architecture/storage.md)
- [Multi-Tenancy](../architecture/multi-tenancy.md)
- [Event & Hook System](../architecture/event-hook-system.md)
- [Configuration](../getting-started/configuration.md)
- [Security Model](../security/security-model.md)
- [CLI Reference](../reference/cli-reference.md)
- [Documentation Index](../README.md)
