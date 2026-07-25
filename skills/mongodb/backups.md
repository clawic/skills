# Backups — Dumps, Snapshots, PITR, and Drills

A backup you have never restored is a hypothesis. Everything below is ordered by how well it survives contact with a real recovery.

## Choosing a Method

| Method | Restores | Right for | Disqualifier |
|---|---|---|---|
| `mongodump`/`mongorestore` | Logical BSON, indexes rebuilt on restore | Small datasets, pre-migration snapshots, moving one collection | Slow at scale, pollutes the WiredTiger cache while dumping, and index rebuild dominates restore time |
| Filesystem / volume snapshot | The data files as they were | Anything above tens of GB; the default for self-hosted | Journal and data must be on the SAME volume, or the snapshot is not crash-consistent |
| Continuous PITR (Atlas, Ops Manager) | Any second within the retention window | Anyone who can answer "restore to just before the bad deploy" | Managed tooling; not something to hand-roll from oplog tailing |
| Sharded cluster backup | All shards plus config servers, at one cluster time | Any sharded cluster | Independent per-shard dumps restore an inconsistent cluster |

## mongodump Done Properly

```bash
mongodump --uri="$URI" --oplog --gzip --archive=dump-$(date +%F).gz
mongorestore --uri="$URI" --oplogReplay --gzip --archive=dump-2026-07-25.gz
```

- `--oplog` captures oplog entries written DURING the dump and `--oplogReplay` applies them on restore: without this pair, a dump of a live database is a set of collections captured at different moments and is not point-in-time consistent.
- `--archive` plus `--gzip` produces a single stream instead of a directory tree — pipeable straight to object storage.
- Restore rebuilds every index from scratch. On a large collection this is most of the restore time; `--numParallelCollections` and `--numInsertionWorkersPerCollection` help, but the honest planning number comes from timing an actual restore.
- `mongodump` does NOT capture users and roles unless you dump the `admin` database too. A restore that works and then rejects every login is this omission (→ `security.md`).
- Dumping from a SECONDARY keeps the read load and cache pollution off the primary. Add `--readPreference=secondary`.

## Filesystem Snapshots

- Requirements: journal on the same volume as data, or `db.fsyncLock()` before the snapshot and `db.fsyncUnlock()` after — and `fsyncLock` blocks writes for the duration, so it belongs on a secondary.
- Restore is a crash-recovery replay: mongod starts, replays the journal, and comes up. Faster than any logical restore because nothing is rebuilt.
- A snapshot from a healthy secondary can seed a new replica set member directly, skipping initial sync — valid only if the snapshot is newer than the oldest oplog entry on the source (→ `replication.md`).
- Encrypted volumes: verify the restore path has the key. A snapshot you cannot decrypt is not a backup, and this is discovered at the worst possible time.

## Point-in-Time Recovery

- PITR = a base snapshot plus continuous oplog capture. The recovery point is any moment covered by both.
- The realistic use is not hardware failure — it is "a migration script deleted the wrong documents at 14:32". Hardware failure is what replication is for.
- Retention is a policy decision with a cost: the oplog archive grows with write volume, not with data size.
- Sharded PITR requires cluster-wide coordination of restore points; this is squarely managed-tooling territory (→ `atlas.md`, `sharding.md`).

## Restoring a Single Collection or Document

- `mongorestore --nsInclude="db.coll"` from a full dump restores one namespace without touching the rest.
- Restore into a SCRATCH database name first (`--nsFrom="db.*" --nsTo="db_restore.*"`), verify, then copy the documents you actually need. Restoring directly over a live collection during an incident is how a partial data loss becomes a total one.
- Recovering a few documents deleted by accident: restore to scratch, `find` them, `insertMany` into the live collection. This should be a rehearsed 10-minute procedure, not an improvisation.

## The Drill

Quarterly, and after any topology change:

1. Restore the latest backup into a scratch cluster — not a subset, the whole thing.
2. Time it end to end, including index rebuild. That number, not the dump duration, is your RTO.
3. Verify: document counts per collection, a checksum-style aggregation on the two most important collections, `db.getUsers()` in `admin`, and `getIndexes()` on the hot collections.
4. Run the application against the restored copy for five minutes.
5. Write down the measured RTO and the date. If it exceeds the RTO the business believes in, that gap is the finding — report it as a number, not as a worry.

## What People Discover During Their First Real Restore

- The restore is slower than the dump, usually by a large factor, because of index rebuilds.
- Users and roles were never backed up (above).
- The backup is from a secondary that was lagging, so the recovery point is older than the timestamp on the file.
- The `admin` database, the config servers, or Atlas Search index definitions were out of scope and have to be recreated by hand (→ `search.md`).
- Nobody knows which of the seven backups in the bucket is the good one, because none of them were ever verified.
- Retention silently expired the backup they needed — retention is set from a policy nobody re-read after the data grew.

## Backup Hygiene

- Store backups on different infrastructure than the database. A snapshot in the same account, deleted by the same credential, protects against exactly one failure mode: disk failure.
- Encrypt at rest and restrict who can read them: a database backup is the database, with none of the access controls (→ `security.md`).
- Alert on backup AGE, not on backup success. A job that succeeds while writing nothing is the common silent failure.
- Keep a copy that a compromised production credential cannot delete. Ransomware against exposed MongoDB deployments is not hypothetical, and the deletion of backups is part of the playbook.
