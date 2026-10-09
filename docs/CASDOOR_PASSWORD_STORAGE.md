# Casdoor password storage and upgrades

The supported Docker Compose authentication deployment uses Argon2id for local
passwords. Casdoor 2.67 creates its built-in organization before reading the
initialization template, so changing the template alone does not secure that
organization or an existing database. The migration below is required for both
new installations and upgrades.

The Casdoor service is private to the Compose network. Its public port is served
by `casdoor-gateway`, which starts only after a read-only database check confirms
strong password storage. A failed check does not modify any account and keeps
the public authentication port closed. Other containers on the deployment
network must remain trusted. This startup check is not continuous monitoring.

## New installation

From `docker/astronAgent`, configure `.env` as usual, then run:

```bash
# Initialize Casdoor privately; its public gateway is not started.
docker compose -f docker-compose-with-auth.yaml up -d casdoor

# Preview counts only. Passwords and hashes are never printed.
docker compose -f docker-compose-with-auth.yaml run --rm casdoor-passwords

# Stop all Casdoor replicas before applying the transaction.
docker compose -f docker-compose-with-auth.yaml stop casdoor
docker compose -f docker-compose-with-auth.yaml run --rm casdoor-passwords --apply

# Start the application and the verified authentication gateway.
docker compose -f docker-compose-with-auth.yaml up -d
```

Wait for Casdoor initialization to finish before previewing. A missing schema
causes the tool to exit without changes; retry the preview after initialization.
The first migration also converts the upstream built-in admin account. Change
the sample administrator password and remove unused sample accounts before
allowing untrusted access. Hashing does not make a published sample password
secret.

The same procedure applies to `docker-compose-with-auth-rpa.yaml` and to the
standalone `docker-compose-auth.yml`; use that filename in every command.

## Existing deployment

1. Back up the **Casdoor MySQL database**, verify that the backup can be restored,
   and arrange an authentication maintenance window. Treat backups as sensitive:
   older backups can contain plaintext passwords.
2. Stop public authentication traffic and **all** Casdoor instances using that
   database, including replicas outside this Compose project. Keep MySQL running.
   Existing sessions may still be valid; stop application access during maintenance.
3. Use the migration preview to review counts and resolve any rejected state.
   Set `CASDOOR_MYSQL_PASSWORD` in `.env` to the existing `casdoor` database user's
   password if it differs from the sample. Changing this variable does not change
   credentials already stored in a persistent MySQL volume.
4. Run `casdoor-passwords --apply`, then run it again without `--apply`; the second
   preview should report zero account and policy updates.
5. Restart Casdoor and recreate the verification job and gateway:

   ```bash
   docker compose -f docker-compose-with-auth.yaml up -d casdoor
   docker compose -f docker-compose-with-auth.yaml up -d --force-recreate casdoor-password-check casdoor-gateway
   ```

6. Check successful and failed logins with a test account before reopening access.
   Keep an administrator session or a verified recovery procedure available until
   validation is complete. Restore the database backup if recovery is necessary;
   Argon2id hashes cannot be converted back to plaintext.

The migration preserves account IDs and login passwords. It converts plaintext
passwords with independently generated salts, preserves existing valid bcrypt
and Argon2id hashes, and pins inherited bcrypt metadata before changing the
organization policy. Casdoor can upgrade preserved bcrypt hashes to Argon2id
after successful login. Repeated migration does not hash existing hashes again.
All password and organization changes commit in one transaction.

The tool intentionally rejects unsupported or malformed hashes, ambiguous
hash-looking plaintext, local passwords on LDAP-linked accounts, and organization
master/default password overrides. Resolve these through Casdoor administration
before retrying. Do not reinterpret a digest as a plaintext password or delete
accounts to bypass a failed check. Passwordless external-identity accounts remain
passwordless. The tool targets the unprefixed MySQL schema of Casdoor **v2.67.0**;
custom table prefixes, other databases, and externally managed identity providers
require their own migration and validation.

Keep `initDataNewOnly = true`. Setting it to false can recreate organizations and
users from the sample template, resetting account data on restart. Template users
retain `passwordType: plain` as **input metadata**: Casdoor's `AddUser` hashes their
input password using the organization's Argon2id policy before storage.

## Scope and verification

The automatic public-port gate applies to Docker Compose. The Helm chart is still
documented as under development; do not expose its Casdoor service without an
equivalent private initialization, database migration, and verification process.
This change does not deploy or modify any production database.

Run local planning tests with Python 3.12 and the listed dependencies:

```bash
python -m pip install -r docker/astronAgent/casdoor/password-storage/requirements.txt
python -m unittest discover -s docker/astronAgent/casdoor/password-storage -p test_passwords.py -v
```

The `Casdoor password storage` workflow also runs the real Casdoor v2.67.0 and
MySQL images in disposable containers. It checks fresh-user hashing, legacy
account migration, failed and successful logins, rollback, idempotence, restart
behavior, and the public gateway's startup gate.

Upstream behavior: [initialization](https://github.com/casdoor/casdoor/blob/v2.67.0/object/init_data.go),
[new-user hashing](https://github.com/casdoor/casdoor/blob/v2.67.0/object/user.go#L933),
[login-time migration](https://github.com/casdoor/casdoor/blob/v2.67.0/object/check.go#L248),
[Argon2id support](https://github.com/casdoor/casdoor/blob/v2.67.0/cred/argon2id.go).
