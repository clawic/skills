# Security — Auth, Roles, Network, Injection, Encryption

MongoDB's historical reputation for insecurity came from one default: early versions bound to all interfaces with authentication off, and tens of thousands of open databases were found and ransomed. Modern versions bind to localhost by default. The exposure now comes from operators who change that binding without turning auth on.

## The Exposure Audit (do this first, every time)

1. `net.bindIp` — is the server listening beyond localhost or a private interface? Binding `0.0.0.0` is legitimate behind a firewall and catastrophic without one.
2. Is `security.authorization: enabled` set? Self-hosted MongoDB does not require authentication unless you turn it on.
3. Does the deployment answer from the public internet? Test from outside your network, not from a machine on it.
4. Is TLS on, and does the client verify the certificate? Unverified TLS is an encrypted channel to whoever answers.
5. Who has `root` or `dbAdminAnyDatabase`? Count the humans and the service accounts separately.
6. Are backups protected as strongly as the database (→ `backups.md`)?

## Authentication

- SCRAM-SHA-256 is the default mechanism (MongoDB >=4.0) and is right for most deployments. SCRAM-SHA-1 remains for legacy drivers and should be retired.
- x.509 certificate auth removes shared secrets entirely — the strongest option for service-to-service, at the cost of certificate lifecycle management.
- AWS IAM and OIDC (MongoDB >=7.0) let identity come from the platform, which removes the "where is the database password stored" question rather than answering it.
- LDAP and Kerberos are Enterprise/Atlas features.
- **Users belong to a database.** A user created in `admin` authenticates with `authSource=admin` regardless of which database it reads. Error 18 with correct credentials is almost always a wrong `authSource` (→ `errors.md`).
- Never share one connection string across services. Per-service users make revocation and audit possible, and they make the `appName`-to-user mapping obvious during an incident.

## Roles

Grant the narrowest built-in role that works, then a custom role if none fits:

| Role | Grants | Use for |
|---|---|---|
| `read` / `readWrite` | On one database | Application service accounts — this is where most accounts belong |
| `dbAdmin` | Indexes, stats, validation on one database; no data reads | Migration and schema tooling |
| `clusterMonitor` | `serverStatus`, `replSetGetStatus`, profiler reads | Monitoring agents (→ `monitoring.md`) |
| `backup` / `restore` | The minimum for dump and restore | Backup jobs |
| `readAnyDatabase` | Reads across everything | Analytics, when scoped reads genuinely cannot work |
| `root` | Everything | Break-glass only, with a rotation record |

- Custom roles compose actions and resources precisely: `{role: "orderReader", privileges: [{resource: {db: "shop", collection: "orders"}, actions: ["find"]}]}`.
- Error 13 names the action and namespace it wanted. Grant that action, not the role that contains it plus forty others (→ `errors.md`).
- Restrict a role to a document filter for tenant isolation where the platform supports it — otherwise enforcement belongs in a data-access layer that cannot be bypassed (→ `schema.md`).

## Network

- Private networking beats an allowlist: VPC peering or private endpoints mean the database has no public address at all. An IP access list is the fallback, not the goal.
- TLS in transit, with client-side certificate verification enabled. A driver configured with `tlsAllowInvalidCertificates` for a one-off debugging session and never changed back is a recurring finding.
- Do not expose the database to developer laptops directly. A bastion or a managed proxy gives you an audit trail and one place to revoke.
- Change streams, `$out`, `$merge`, and `$lookup` all run server-side: network egress rules on the application do not constrain what the database can reach.

## Injection

MongoDB is not immune because it is not SQL. The attack is an OBJECT arriving where a scalar was expected.

```javascript
// Vulnerable: req.body.username may be {"$ne": null}
await users.findOne({username: req.body.username, password: req.body.password});
```

- With `{"$ne": null}` in both fields, that query returns the first user in the collection and the login succeeds.
- Fix by type, not by escaping: assert `typeof username === "string"` before it reaches the query, or validate the request body against a schema. Mongoose's casting does this incidentally — and that protection vanishes the moment code drops to the raw driver (→ `connections.md`).
- Reject keys beginning with `$` and containing `.` in any user-controlled object before it becomes a filter or an update document.
- `$where`, `$function`, `$accumulator`, and `mapReduce` execute JavaScript server-side: user input reaching any of them is remote code execution inside the database process. Disable server-side JavaScript (`security.javascriptEnabled: false`) if nothing legitimately needs it — most deployments do not.
- Aggregation pipelines assembled from user input are the same risk one level up: a user-supplied `$lookup` or `$out` stage reads or writes collections the endpoint never intended.

## Encryption at Rest and In Use

- **Volume encryption** protects against a stolen disk and nothing else. It is table stakes, not a control against an attacker with database access.
- **Encrypted storage engine** (Enterprise/Atlas) encrypts the data files with keys in a KMS.
- **CSFLE — Client-Side Field Level Encryption** (MongoDB >=4.2): the driver encrypts specific fields before they leave the application. The server stores and returns ciphertext and cannot read the values, which is what makes it meaningful against a compromised database. Deterministic encryption allows equality matching on the field; randomized encryption is stronger but the field becomes unqueryable.
- **Queryable Encryption** (GA in MongoDB >=7.0) extends this to equality and range queries over encrypted fields without the server seeing plaintext.
- Both add a key-management dependency and change what you can index. Decide field by field: encrypt what a breach would make catastrophic, not everything.

## Auditing and Hygiene

- Enable auditing (Enterprise/Atlas) for authentication events, role changes, and DDL at minimum. Query-level auditing is expensive; the administrative events are the ones investigations need.
- Rotate credentials on a schedule and on every departure. A rotation you have never rehearsed will fail during the incident that forces it.
- Log the connection's `appName` and user together so an unusual access pattern names a service (→ `monitoring.md`).
- Keep server versions current: security fixes ship in patch releases and the upgrade path is a rolling restart (→ `replication.md`).
- Never put credentials in a connection string committed to a repository, in a shell command that lands in history, or in a `mongosh` invocation (→ `mongosh.md`).
