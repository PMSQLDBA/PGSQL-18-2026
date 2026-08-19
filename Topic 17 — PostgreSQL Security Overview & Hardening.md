# Topic 17 — PostgreSQL Security Overview & Hardening

PostgreSQL security is not one single feature. 

It is a set of controls working together: **authentication, authorization, roles/RBAC, least privilege, network access, TLS, encryption, auditing, secrets management, and operating-system/database hardening**.

A useful way to think about it is:

**Who can connect → From where → How they prove identity → What they can access → What they can do → How data is protected → How activity is audited.**

---

## 1. PostgreSQL Security Architecture

PostgreSQL security can be viewed in layers:

```text
Application / User
       |
       v
+-------------------------+
| Network Security        |
| Firewall / Private Net  |
| TLS                     |
+-------------------------+
       |
       v
+-------------------------+
| pg_hba.conf             |
| Who can connect?        |
| From where?             |
| Authentication method   |
+-------------------------+
       |
       v
+-------------------------+
| Authentication          |
| SCRAM / Certificate     |
| LDAP / Kerberos etc.    |
+-------------------------+
       |
       v
+-------------------------+
| PostgreSQL Roles        |
| LOGIN / NOLOGIN         |
| Role Membership         |
+-------------------------+
       |
       v
+-------------------------+
| Authorization           |
| CONNECT / USAGE         |
| SELECT / INSERT / etc.  |
+-------------------------+
       |
       v
+-------------------------+
| Object Security         |
| DB / Schema / Table     |
| Sequence / Function     |
| Row Level Security      |
+-------------------------+
       |
       v
+-------------------------+
| Audit / Monitoring      |
| Logs / pgAudit / SIEM   |
+-------------------------+
```

The important point is that `pg_hba.conf` does **not** decide what a user can do inside the database. It primarily controls **whether and how a client can authenticate**.

Privileges and roles determine what an authenticated session can do.

---

# 2. Authentication vs Authorization

These two concepts are commonly confused.

### Authentication

Authentication answers:

> **Who are you?**

Examples include:

* SCRAM password authentication
* client certificates
* LDAP
* Kerberos/GSSAPI
* operating-system-based authentication

For example:

```text
User: app_user
Password: ********
```

PostgreSQL verifies whether the client really is `app_user`.

### Authorization

Authorization answers:

> **What are you allowed to do?**

For example:

```sql
GRANT SELECT ON TABLE orders TO reporting_role;
```

The user may successfully authenticate but still receive:

```text
ERROR: permission denied for table orders
```

That means authentication succeeded but authorization denied the operation.

---

# 3. PostgreSQL Users and Roles

PostgreSQL uses a unified **role** model.

Historically people talk about "users" and "groups", but technically PostgreSQL manages both through roles.

A role with `LOGIN` behaves like a database user.

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'password';
```

A role without `LOGIN` is commonly used as a permission/group role.

```sql
CREATE ROLE app_readonly
NOLOGIN;
```

Then:

```sql
GRANT app_readonly TO app_user;
```

Conceptually:

```text
Role
 |
 +-- LOGIN? Yes  -> User/login role
 |
 +-- LOGIN? No   -> Group/permission role
```

You can inspect roles using:

```sql
\du
```

or:

```sql
SELECT rolname,
       rolsuper,
       rolcreatedb,
       rolcreaterole,
       rolcanlogin
FROM pg_roles;
```

---

# 4. RBAC — Role-Based Access Control

A good production design usually avoids assigning large numbers of permissions directly to individual login users.

Instead:

```text
Privileges
    |
    v
Permission Roles
    |
    v
Login Roles
```

Example:

```sql
CREATE ROLE app_readonly NOLOGIN;
CREATE ROLE app_readwrite NOLOGIN;
CREATE ROLE app_admin NOLOGIN;
```

Assign privileges:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA app
TO app_readonly;
```

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO app_readwrite;
```

Then create login roles:

```sql
CREATE ROLE john LOGIN PASSWORD '...';
CREATE ROLE api_service LOGIN PASSWORD '...';
```

Assign membership:

```sql
GRANT app_readonly TO john;

GRANT app_readwrite TO api_service;
```

Now permissions are easier to manage.

```text
john
 |
 +---- app_readonly
          |
          +---- SELECT

api_service
 |
 +---- app_readwrite
          |
          +---- SELECT
          +---- INSERT
          +---- UPDATE
          +---- DELETE
```

This is much cleaner than individually granting privileges to hundreds of users.

---

# 5. Least Privilege

The **principle of least privilege** means:

> Give an identity only the permissions required to perform its job.

For example, an application probably needs:

```text
SELECT
INSERT
UPDATE
DELETE
```

It usually does **not** need:

```text
SUPERUSER
CREATEDB
CREATEROLE
ALTER SYSTEM
DROP DATABASE
```

A bad design would be:

```sql
CREATE ROLE application
LOGIN
SUPERUSER
PASSWORD '...';
```

If the application is compromised, the attacker effectively owns the PostgreSQL cluster.

A safer design is:

```sql
CREATE ROLE application LOGIN;
```

Then grant only the required privileges.

---

# 6. PostgreSQL Superuser

PostgreSQL superusers are extremely powerful.

Example:

```sql
CREATE ROLE dba_admin
LOGIN
SUPERUSER;
```

A superuser can bypass almost all normal database permission checks.

Check superusers:

```sql
SELECT rolname
FROM pg_roles
WHERE rolsuper;
```

Production systems should keep the number of superuser roles very small.

A typical model is:

```text
                PostgreSQL
                    |
          +---------+---------+
          |                   |
     DBA/Admin            Applications
     Superuser            Non-superuser
                              |
                       Limited privileges
```

Application accounts should not normally be superusers.

---

# 7. Important Role Attributes

PostgreSQL roles can have powerful attributes.

```sql
CREATE ROLE example
LOGIN
NOSUPERUSER
NOCREATEDB
NOCREATEROLE
NOREPLICATION;
```

Important attributes include:

| Attribute          | Meaning                                      |
| ------------------ | -------------------------------------------- |
| `LOGIN`            | Can establish database sessions              |
| `SUPERUSER`        | Bypasses most permission checks              |
| `CREATEDB`         | Can create databases                         |
| `CREATEROLE`       | Can create/manage roles within its authority |
| `REPLICATION`      | Can initiate replication connections         |
| `BYPASSRLS`        | Can bypass Row-Level Security                |
| `CONNECTION LIMIT` | Limits concurrent connections                |

For example:

```sql
ALTER ROLE api_user CONNECTION LIMIT 50;
```

---

# 8. PostgreSQL Privilege Hierarchy

PostgreSQL permissions exist at several levels.

```text
Cluster
 |
 +-- Database
      |
      +-- Schema
           |
           +-- Table
           |
           +-- Sequence
           |
           +-- Function / Procedure
```

A user may therefore need permissions at more than one level.

For example:

```sql
GRANT CONNECT ON DATABASE salesdb TO app_user;
```

Then:

```sql
GRANT USAGE ON SCHEMA sales TO app_user;
```

Then:

```sql
GRANT SELECT ON TABLE sales.orders TO app_user;
```

Having `SELECT` on a table does not automatically solve every surrounding database/schema permission requirement.

---

# 9. Database Privileges

Common database-level privileges include:

```text
CONNECT
CREATE
TEMPORARY
```

Example:

```sql
GRANT CONNECT ON DATABASE production TO application;
```

You can remove access:

```sql
REVOKE CONNECT ON DATABASE production FROM application;
```

A common hardening step is reviewing what `PUBLIC` can do.

For example:

```sql
REVOKE CONNECT ON DATABASE production FROM PUBLIC;
```

Then explicitly allow approved roles:

```sql
GRANT CONNECT ON DATABASE production TO app_users;
```

---

# 10. Schema Security

Schemas are namespaces containing database objects.

Example:

```text
sales
 |
 +-- customers
 +-- orders
 +-- payments
```

Two important schema privileges are:

```text
USAGE
CREATE
```

`USAGE` allows access to objects in the schema when the user also has the appropriate object privileges.

```sql
GRANT USAGE ON SCHEMA sales TO reporting_role;
```

`CREATE` allows creation of objects inside the schema.

```sql
GRANT CREATE ON SCHEMA sales TO developer_role;
```

Production application users normally should not receive `CREATE` unless required.

---

# 11. Table Privileges

Common table privileges include:

```text
SELECT
INSERT
UPDATE
DELETE
TRUNCATE
REFERENCES
TRIGGER
```

Example:

```sql
GRANT SELECT ON sales.orders TO analyst_role;
```

Read/write:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON sales.orders
TO application_role;
```

Remove a privilege:

```sql
REVOKE DELETE
ON sales.orders
FROM application_role;
```

---

# 12. Sequence Permissions

Sequences are often forgotten during permission configuration.

Suppose a table uses an identity/sequence-backed value.

The application may have:

```sql
GRANT INSERT ON sales.orders TO application_role;
```

but still encounter sequence-related permission problems depending on how IDs/defaults are implemented.

Sequence privileges include:

```text
USAGE
SELECT
UPDATE
```

Example:

```sql
GRANT USAGE
ON SEQUENCE sales.orders_id_seq
TO application_role;
```

This is an important troubleshooting area when INSERT permissions appear correct but generated IDs cause failures.

---

# 13. Default Privileges

One common production problem is:

> Existing tables work, but newly created tables are inaccessible.

For example:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA sales
TO reporting_role;
```

This affects existing tables.

It does not automatically guarantee the same permission on every future table.

Use default privileges:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA sales
GRANT SELECT ON TABLES
TO reporting_role;
```

A critical detail: **default privileges are associated with the role that creates future objects**.

If migrations run as `migration_user`, configure the defaults for that owner:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE migration_user
IN SCHEMA sales
GRANT SELECT ON TABLES
TO reporting_role;
```

This detail causes many real production permission issues.

---

# 14. PUBLIC Role

Every PostgreSQL role implicitly belongs to the special pseudo-role:

```text
PUBLIC
```

Therefore a grant to `PUBLIC` effectively applies broadly.

Always review permissions such as:

```sql
GRANT ... TO PUBLIC;
```

Depending on the environment, hardening may include revoking unnecessary default/public access and then explicitly granting required privileges.

Do this carefully because PostgreSQL defaults and behavior can vary by object type and version.

---

# 15. pg_hba.conf

`pg_hba.conf` stands for:

> **PostgreSQL Host-Based Authentication**

It controls:

```text
Connection type
Database
User
Client source
Authentication method
```

Typical format:

```text
TYPE  DATABASE  USER  ADDRESS          METHOD
```

Example:

```text
host  production  app_user  10.20.0.0/16  scram-sha-256
```

Meaning:

```text
TCP connection
       +
production database
       +
app_user
       +
client from 10.20.0.0/16
       +
authenticate using SCRAM
```

---

# 16. pg_hba.conf Rule Ordering

This is extremely important:

> PostgreSQL uses the **first matching `pg_hba.conf` record**.

It does not search all matching rules and choose the "most secure" one.

Consider:

```text
host all all 0.0.0.0/0 scram-sha-256
host production app_user 10.20.1.0/24 reject
```

The broad first rule may match before PostgreSQL reaches the intended rejection rule.

Rule order therefore matters.

Think of it like a firewall rule list:

```text
Connection
    |
    v
Rule 1 matches? ---- yes ---> use Rule 1
    |
    no
    v
Rule 2 matches? ---- yes ---> use Rule 2
    |
    no
    v
...
```

---

# 17. Common Authentication Methods

You may encounter:

```text
trust
reject
scram-sha-256
md5
peer
cert
ldap
radius
gss
sspi
pam
```

For modern password-based deployments, **SCRAM-SHA-256** is generally preferred over older MD5-based password authentication.

Example:

```text
hostssl production app_user 10.20.0.0/16 scram-sha-256
```

---

# 18. Why `trust` Is Dangerous

Example:

```text
host all all 0.0.0.0/0 trust
```

This effectively says:

```text
Any source
   |
Any database
   |
Any PostgreSQL role
   |
No password authentication
```

That is extremely dangerous for remotely reachable production systems.

`trust` may occasionally be appropriate for very controlled local/test scenarios, but it should not be casually used.

---

# 19. SCRAM-SHA-256

SCRAM is PostgreSQL's modern password authentication mechanism.

Typical configuration:

```text
password_encryption = 'scram-sha-256'
```

Then:

```text
hostssl all all 10.20.0.0/16 scram-sha-256
```

Important distinction:

```text
password_encryption
```

controls how newly set PostgreSQL passwords/verifiers are stored.

`pg_hba.conf` controls how clients authenticate.

These are related but separate settings.

---

# 20. TLS/SSL

Without transport encryption, database traffic may be exposed to network interception depending on the network path.

TLS protects:

```text
Application
    |
    | encrypted connection
    v
PostgreSQL
```

rather than:

```text
Application
    |
    | plaintext traffic
    v
PostgreSQL
```

Server configuration typically involves settings such as:

```text
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

Exact paths and certificate-management methods depend on the platform.

---

# 21. `host` vs `hostssl`

In `pg_hba.conf`:

```text
host
```

can match TCP/IP connections without requiring TLS.

```text
hostssl
```

matches only SSL/TLS connections.

For example:

```text
hostssl production app_user 10.20.0.0/16 scram-sha-256
```

can be part of a policy requiring encrypted remote connections.

---

# 22. Client Certificate Authentication

PostgreSQL can also use TLS client certificates.

Conceptually:

```text
Client
 |
 | presents certificate
 v
PostgreSQL
 |
 | validates certificate
 v
Authentication
```

This can provide stronger machine identity than password-only authentication when properly deployed.

Certificate lifecycle management then becomes important:

```text
Issuance
Renewal
Revocation
Expiration monitoring
Private-key protection
```

---

# 23. Encryption in Transit

Encryption in transit protects data moving between systems.

Examples:

```text
Application <--- TLS ---> PostgreSQL

DBA Client   <--- TLS ---> PostgreSQL

Replica      <--- TLS ---> Primary
```

This helps protect:

* credentials
* SQL statements
* query results
* application data

TLS configuration should also validate the server's identity rather than merely encrypting traffic.

---

# 24. Encryption at Rest

PostgreSQL security discussions should distinguish **encryption in transit** from **encryption at rest**.

At-rest encryption is commonly implemented outside core PostgreSQL storage using infrastructure such as:

```text
Encrypted disk
Encrypted volume
Cloud storage encryption
Encrypted backups
KMS-managed keys
```

Architecture:

```text
PostgreSQL
    |
Data files
    |
Encrypted storage
    |
Physical disk
```

Also remember:

> Database encryption and backup encryption are separate concerns.

A secure database with unprotected backup files is still a security problem.

---

# 25. Column-Level / Application-Level Encryption

Highly sensitive values may require additional protection.

Examples:

```text
SSN
Bank information
Sensitive tokens
Personal identifiers
```

Possible architecture:

```text
Application
    |
Encrypt value
    |
PostgreSQL stores ciphertext
```

or controlled database-side encryption using appropriate PostgreSQL capabilities/extensions.

The important security question becomes:

> Where are the encryption keys stored?

Storing ciphertext and its unrestricted key in the same security boundary significantly reduces the benefit.

---

# 26. Secrets Management

Applications need credentials to connect to PostgreSQL.

Avoid putting production passwords directly in:

```text
Source code
Git repositories
Docker images
Shell scripts
Wiki pages
CI/CD logs
```

Bad:

```python
password = "ProductionPassword123"
```

A better architecture uses a secret-management system:

```text
Application
     |
     v
Secret Manager / Vault
     |
     v
Database credentials
     |
     v
PostgreSQL
```

Examples include cloud secret managers, enterprise vault systems, or Kubernetes-integrated secret-management solutions.

---

# 27. Secret Rotation

Database credentials should have a lifecycle.

```text
Create
  |
Use
  |
Rotate
  |
Revoke old credential
  |
Audit
```

Long-lived passwords increase exposure.

Service credentials should ideally support:

* automated rotation
* controlled retrieval
* limited lifetime where practical
* auditability
* least privilege

---

# 28. Avoid Shared Accounts

Avoid architectures where many people use:

```text
dba_shared
```

with one password.

Why?

Suppose:

```text
DROP TABLE customer;
```

appears in logs.

If 15 DBAs use the same account, identifying who performed the action becomes difficult.

Prefer:

```text
alice_admin
bob_admin
charlie_admin
```

or centralized identity integration.

This provides better accountability.

---

# 29. Service Accounts

Each application or service should preferably have its own identity.

Instead of:

```text
shared_app_user
     |
     +-- Billing
     +-- Orders
     +-- Inventory
```

prefer:

```text
billing_service
orders_service
inventory_service
```

Then permissions can differ:

```text
billing_service
   -> billing schema

orders_service
   -> orders schema

inventory_service
   -> inventory schema
```

This limits the blast radius if one service is compromised.

---

# 30. Row-Level Security

Normal table permissions answer:

> Can this user query this table?

Row-Level Security can answer:

> Which rows inside this table can this user access?

Example:

```text
orders
+----+-----------+--------
id   tenant_id   amount
1    A           100
2    B           200
3    A           300
```

Tenant A should see:

```text
1 A 100
3 A 300
```

but not:

```text
2 B 200
```

Enable RLS:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

Then define policies appropriate for the application's identity/tenant model.

RLS is particularly useful in multi-tenant architectures, but policies must be designed and tested carefully.

---

# 31. Function Security

Functions can introduce privilege-escalation risks.

Pay particular attention to:

```text
SECURITY INVOKER
SECURITY DEFINER
```

`SECURITY INVOKER` generally runs with the calling user's privileges.

`SECURITY DEFINER` runs with the function owner's privileges.

Therefore a poorly designed `SECURITY DEFINER` function can become a privilege-escalation path.

Special care is required around:

* function ownership
* executable privileges
* object qualification
* `search_path`
* dynamic SQL
* input validation

---

# 32. `search_path` Security

PostgreSQL resolves unqualified object names using `search_path`.

For example:

```sql
SELECT process_payment();
```

PostgreSQL needs to determine which `process_payment()` function is meant.

In privileged code, insecure object resolution can create security problems if an attacker can create an object with a matching name in a schema searched earlier.

For security-sensitive functions, explicitly control object resolution and avoid trusting writable schemas.

---

# 33. Ownership vs Privileges

Ownership is stronger than normal grants.

The owner of an object has significant control over that object.

Example:

```text
Owner: migration_owner
Users: application_role
```

A good pattern can be:

```text
migration_owner
      |
      +---- owns tables

application_role
      |
      +---- receives required grants
```

The application therefore uses objects without owning them.

This prevents the runtime application identity from casually altering or dropping its own production schema objects.

---

# 34. Separate Runtime and Migration Roles

A strong application design separates:

```text
Migration Role
Runtime Role
```

For example:

```text
migration_user
 |
 +-- CREATE TABLE
 +-- ALTER TABLE
 +-- CREATE INDEX
 +-- schema changes
```

while:

```text
application_user
 |
 +-- SELECT
 +-- INSERT
 +-- UPDATE
 +-- DELETE
```

This means compromising the application does not automatically give an attacker schema-management permissions.

---

# 35. Logging and Auditing

Security controls are incomplete without visibility.

PostgreSQL can log important events and statements depending on configuration.

Relevant configuration areas include settings such as:

```text
log_connections
log_disconnections
log_statement
log_duration
log_min_duration_statement
```

However, enabling excessive logging can:

* create huge log volumes
* expose sensitive SQL values
* increase storage requirements
* impact performance

Logging policy should therefore balance security, privacy, operational value, and cost.

---

# 36. pgAudit

For environments requiring stronger database auditing, the **pgAudit** extension is commonly used.

It can help provide structured auditing around categories such as:

```text
READ
WRITE
DDL
ROLE
FUNCTION
MISC
```

A typical enterprise flow might be:

```text
PostgreSQL
    |
    v
PostgreSQL / pgAudit logs
    |
    v
Central log platform
    |
    v
SIEM
    |
    v
Alerts / Investigation
```

Auditing should answer questions such as:

```text
Who connected?
When?
From where?
What object was accessed?
What privilege/DDL change occurred?
Was authentication unsuccessful?
```

---

# 37. Failed Login Monitoring

Repeated authentication failures can indicate:

* wrong application credentials
* expired/rotated secrets
* configuration errors
* brute-force attempts
* unauthorized access attempts

A monitoring pipeline might be:

```text
PostgreSQL logs
     |
     v
Log collector
     |
     v
SIEM
     |
     +---- Detect repeated failures
     |
     +---- Alert security team
```

Database security should integrate with the wider organization's monitoring platform.

---

# 38. Network Hardening

PostgreSQL should normally not be unnecessarily exposed to the public internet.

Preferred architecture:

```text
Internet
   |
Load Balancer/API
   |
Application
   |
Private Network
   |
PostgreSQL
```

Rather than:

```text
Internet
   |
PostgreSQL :5432
```

Controls may include:

* private subnets
* security groups/firewalls
* controlled routing
* VPN/private connectivity
* bastion/access gateways where appropriate
* restrictive `pg_hba.conf`

Defense in depth means using multiple layers rather than relying only on PostgreSQL authentication.

---

# 39. `listen_addresses`

PostgreSQL's:

```text
listen_addresses
```

controls the network interfaces on which PostgreSQL accepts TCP connections.

For example:

```text
listen_addresses = '*'
```

means PostgreSQL listens broadly on available interfaces.

That does **not automatically mean the database is publicly accessible**, because firewalls, routing and `pg_hba.conf` still matter.

But production systems should avoid unnecessarily broad exposure.

---

# 40. Port 5432

The default PostgreSQL TCP port is:

```text
5432
```

Changing the port may reduce trivial scanning/noise, but it is **not a meaningful replacement for security controls**.

Real security comes from:

```text
Network isolation
Firewall rules
pg_hba.conf
Strong authentication
TLS
Least privilege
Monitoring
```

not from relying on a non-default port.

---

# 41. Patch Management

Security hardening includes keeping PostgreSQL supported and patched.

A lifecycle should include:

```text
Security advisory
      |
      v
Impact assessment
      |
      v
Test patch
      |
      v
Backup / rollback planning
      |
      v
Production deployment
      |
      v
Validation
```

Also patch related components:

* operating system
* PostgreSQL extensions
* connection poolers
* backup software
* drivers
* monitoring agents

---

# 42. Extension Security

Extensions run inside the PostgreSQL environment and can significantly expand capabilities.

Check installed extensions:

```sql
SELECT *
FROM pg_extension;
```

Only install approved extensions.

Security review should consider:

```text
Who maintains it?
Is it patched?
Does it require superuser?
Does it load native code?
What privileges does it introduce?
Is it still required?
```

Unused extensions should be reviewed and removed where appropriate.

---

# 43. Backup Security

Backups contain production data and must be protected as carefully as the primary database.

Security requirements include:

```text
Encryption
Access control
Retention
Immutable/offline protection where appropriate
Audit logging
Secure deletion
Restore testing
```

Architecture:

```text
PostgreSQL
    |
    v
Backup
    |
    +--> Encrypted
    +--> Restricted access
    +--> Separate account/location
    +--> Retention policy
    +--> Restore tested
```

A database with excellent security but publicly accessible backup files is not secure.

---

# 44. Replication Security

Replication connections can expose large amounts of database data.

Replication identities should therefore be tightly controlled.

Example:

```sql
CREATE ROLE replication_user
LOGIN
REPLICATION;
```

Restrict its network access in `pg_hba.conf`.

For example:

```text
hostssl replication replication_user 10.20.5.10/32 scram-sha-256
```

Conceptually:

```text
Primary
   |
 encrypted + authenticated
   |
Replica
```

Replication credentials should not be reused by normal applications.

---

# 45. Security Hardening Baseline

A practical production PostgreSQL baseline should include:

```text
[Network]
✓ Database on private network
✓ Firewall restricts port 5432
✓ Only approved application/admin sources allowed

[Authentication]
✓ SCRAM or stronger enterprise authentication
✓ No remote trust authentication
✓ Individual admin identities
✓ Separate application identities

[Authorization]
✓ Least privilege
✓ RBAC/group roles
✓ Minimal superusers
✓ Review PUBLIC privileges
✓ Separate runtime/migration roles

[Transport]
✓ TLS enabled
✓ Certificate validation
✓ Certificates rotated

[Data]
✓ Storage encryption
✓ Backup encryption
✓ Sensitive-field strategy where required

[Secrets]
✓ No hardcoded passwords
✓ Central secret management
✓ Credential rotation

[Auditing]
✓ Connection logging
✓ Failed-authentication monitoring
✓ Privileged activity auditing
✓ Centralized log/SIEM integration

[Operations]
✓ PostgreSQL patched
✓ OS patched
✓ Extensions reviewed
✓ Backup restore tested
✓ Access periodically reviewed
```

---

# 46. Production Security Architecture Example

A mature environment might look like this:

```text
                    Users / Applications
                            |
                    Identity / Secrets
                            |
                            v
                 +---------------------+
                 | Application Layer   |
                 +---------------------+
                            |
                         TLS
                            |
                            v
                  Firewall / Private NW
                            |
                            v
                 +---------------------+
                 | PostgreSQL          |
                 |                     |
                 | pg_hba.conf         |
                 | SCRAM / Cert/Auth   |
                 | Roles + RBAC        |
                 | Least Privilege     |
                 | RLS if required     |
                 +---------------------+
                      |            |
                      |            |
               Encrypted        Audit Logs
                Storage             |
                      |              v
                      |             SIEM
                      |
                Encrypted Backup
                      |
                Restricted Storage
```

This is **defense in depth**.

If one security layer fails, another layer still reduces the attacker's options.

---

# 47. Common Security Mistakes

Some of the most common PostgreSQL security problems are:

| Mistake                      | Risk                                 |
| ---------------------------- | ------------------------------------ |
| Application uses `SUPERUSER` | Full cluster compromise              |
| `trust` for remote clients   | Authentication bypass                |
| Database exposed publicly    | Increased attack surface             |
| Shared DBA accounts          | Poor accountability                  |
| Hardcoded passwords          | Credential leakage                   |
| No TLS                       | Network interception                 |
| Excessive `PUBLIC` grants    | Unintended access                    |
| Runtime app owns schema      | Schema modification risk             |
| No default privileges        | New objects get incorrect access     |
| Unencrypted backups          | Data exposure                        |
| No audit logging             | Weak incident investigation          |
| Long-lived secrets           | Larger credential exposure window    |
| Broad `pg_hba.conf` networks | Unnecessary connection surface       |
| Unpatched PostgreSQL         | Known vulnerabilities remain exposed |

---

# 48. DBA Troubleshooting — Permission Denied

Suppose:

```text
ERROR: permission denied for table orders
```

A DBA should work through the access path rather than immediately granting privileges.

First identify the current identity:

```sql
SELECT current_user, session_user;
```

Check role membership:

```sql
\du
```

Check schema permissions:

```sql
\dn+
```

Check table privileges:

```sql
\dp sales.orders
```

You can also test privileges:

```sql
SELECT has_table_privilege(
    'app_user',
    'sales.orders',
    'SELECT'
);
```

Check schema access:

```sql
SELECT has_schema_privilege(
    'app_user',
    'sales',
    'USAGE'
);
```

The troubleshooting flow is:

```text
Can client reach PostgreSQL?
        |
        v
Does pg_hba.conf match correctly?
        |
        v
Does authentication succeed?
        |
        v
Can role CONNECT to database?
        |
        v
Does role have schema USAGE?
        |
        v
Does role have object privilege?
        |
        v
Is RLS affecting the operation?
        |
        v
Is role membership/SET ROLE involved?
```

Do **not** solve every permission problem with:

```sql
GRANT ALL ...
```

Find the missing permission and grant only what is required.

---

# 49. Interview / Architecture Answer

If asked:

> **How would you secure a PostgreSQL production database?**

A strong answer is:

> I would use defense in depth. PostgreSQL would run on a private network with firewall rules limiting access to approved application and administration sources. Remote connections would use TLS, and `pg_hba.conf` would restrict databases, roles, source networks, and authentication methods. I would use SCRAM or enterprise identity/certificate authentication rather than insecure methods such as remote `trust`.
>
> For authorization, I would implement RBAC with separate login and permission roles and follow least privilege. Application runtime accounts would not be superusers or schema owners, and migration privileges would be separated from runtime privileges.
>
> Credentials would come from a centralized secrets-management platform and be rotated rather than hardcoded. Storage and backups would be encrypted and access-controlled.
>
> Finally, I would enable appropriate connection, authentication, privileged-operation, and audit logging, send security events to centralized monitoring/SIEM, regularly review privileged accounts and `PUBLIC` access, patch PostgreSQL and its extensions, and test backup restoration.

That covers the main architecture areas:

**Network → Authentication → Authorization → Encryption → Secrets → Auditing → Operational hardening.**

---

## Key Commands to Remember

```sql
-- Roles
\du

-- Create login
CREATE ROLE app_user LOGIN PASSWORD '...';

-- Create permission role
CREATE ROLE app_readonly NOLOGIN;

-- Role membership
GRANT app_readonly TO app_user;

-- Database access
GRANT CONNECT ON DATABASE production TO app_readonly;

-- Schema access
GRANT USAGE ON SCHEMA sales TO app_readonly;

-- Table access
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO app_readonly;

-- Future tables
ALTER DEFAULT PRIVILEGES
IN SCHEMA sales
GRANT SELECT ON TABLES TO app_readonly;

-- Check table permissions
\dp sales.orders

-- Check privilege
SELECT has_table_privilege(
    'app_user',
    'sales.orders',
    'SELECT'
);

-- Review superusers
SELECT rolname
FROM pg_roles
WHERE rolsuper;

-- Review extensions
SELECT *
FROM pg_extension;

-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

### Core takeaway

For production PostgreSQL, remember this chain:

```text
Private Network
      ↓
Firewall
      ↓
TLS
      ↓
pg_hba.conf
      ↓
Strong Authentication
      ↓
Individual Login Roles
      ↓
RBAC / Permission Roles
      ↓
Least Privilege
      ↓
Database → Schema → Object Permissions
      ↓
RLS where required
      ↓
Encrypted Storage + Backups
      ↓
Secrets Management
      ↓
Logging + Auditing + SIEM
      ↓
Patching + Regular Access Reviews
```

**`pg_hba.conf` controls who can attempt to enter and how they authenticate. Roles and grants control what they can do after entering. TLS protects data while moving. Storage/backup encryption protects data at rest. Auditing tells you what happened. Least privilege limits the damage when something goes wrong.**
