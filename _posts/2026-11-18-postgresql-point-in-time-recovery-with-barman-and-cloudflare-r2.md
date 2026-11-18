---
title: "PostgreSQL Point-in-Time Recovery with a Barman Sidecar, Docker Compose, and Cloudflare R2"
---

This is the alternative to the [pgBackRest setup](/posts/postgresql-point-in-time-recovery-with-pgbackrest-and-cloudflare-r2/) for anyone who wants point-in-time recovery (PITR) while keeping the stock `postgres:18` image. [Barman](https://pgbarman.org/) runs in its own container and pulls from Postgres over the network, so the database service is not rebuilt and needs no restart.

The trade-off is a second service to run and a more manual restore. Rehearse both restore paths at the end on a scratch machine before you depend on them.

## How it fits together

Barman takes base backups with `pg_basebackup` and receives WAL continuously through a replication slot. Two hook scripts then relay each backup and each WAL file to R2, the pattern the [Barman cloud documentation](https://docs.pgbarman.org/release/3.17.0/user_guide/barman_cloud.html) describes.

The copy inside the sidecar is seconds behind Postgres but lives on the same VPS. The copy in R2 is about two minutes behind.

## Before you start

### Assumptions

- The Compose service is called `db` and runs `postgres:18`.
- The data volume is called `pgdata` and is mounted at `/var/lib/postgresql`, the default for the 18 image, so the data directory is `/var/lib/postgresql/18/docker` ([Docker Hub](https://hub.docker.com/_/postgres)).
- The database port is not published to the internet.
- Commands run from the Compose project directory on the VPS.

### Placeholders

| Placeholder | Meaning |
| --- | --- |
| `<superuser>` | The value of `POSTGRES_USER` in your Compose file (`postgres` if you never set it) |
| `<ACCOUNT_ID>` | Your Cloudflare account ID, as shown in the R2 S3 API endpoint |
| `<BUCKET>` | The R2 bucket that holds the backups |
| `<R2_KEY_ID>` and `<R2_SECRET>` | Access Key ID and Secret Access Key of the R2 API token |
| `<BARMAN_PW>` and `<STREAM_PW>` | Passwords you choose for the two Barman roles |

### Prepare R2

1. Create a bucket used only for these backups, for example `pg-backups`.
2. Create an R2 API token with Object Read & Write permission scoped to that bucket, and note the Access Key ID and Secret Access Key.
3. Do not add a lifecycle rule that deletes objects in this bucket. Deleting WAL files underneath the backup tool breaks restores.

## Setup

### Step 1: Create the two roles

```bash
docker compose exec db psql -U <superuser> -d postgres -c \
  "CREATE ROLE barman LOGIN SUPERUSER PASSWORD '<BARMAN_PW>';"
docker compose exec db psql -U <superuser> -d postgres -c \
  "CREATE ROLE streaming_barman LOGIN REPLICATION PASSWORD '<STREAM_PW>';"
```

### Step 2: Allow replication connections and cap slot growth

The stock image accepts replication connections from localhost only. Add one rule and reload; no restart is needed.

```bash
docker compose exec -u postgres db bash -c \
  'echo "host replication streaming_barman all scram-sha-256" >> "$PGDATA/pg_hba.conf"'
docker compose exec db psql -U <superuser> -d postgres -c \
  "ALTER SYSTEM SET max_slot_wal_keep_size = '2GB';"
docker compose exec db psql -U <superuser> -d postgres -c "select pg_reload_conf();"
```

`max_slot_wal_keep_size` matters: if the sidecar is down, the replication slot makes Postgres hoard WAL, and without a cap the disk fills.

### Step 3: Add the sidecar service

The build starts from `postgres:18` only to get matching `pg_basebackup` and `pg_receivewal` binaries. No database runs in it.

```yaml
services:
  barman:
    build:
      dockerfile_inline: |
        FROM postgres:18
        RUN apt-get update \
         && apt-get install -y --no-install-recommends barman barman-cli-cloud python3-boto3 ca-certificates \
         && rm -rf /var/lib/apt/lists/*
        USER barman
        ENTRYPOINT []
        CMD ["bash", "-c", "while true; do barman cron; barman switch-wal main >/dev/null 2>&1; sleep 60; done"]
    image: local/barman:18
    restart: unless-stopped
    env_file:
      - .env.barman
    volumes:
      - barman_data:/var/lib/barman
      - ./barman-main.conf:/etc/barman.d/main.conf:ro
      - pgdata:/pgdata              # your Postgres data volume, used only for restores
    depends_on:
      - db

volumes:
  barman_data:
```

The loop does two things every minute: `barman cron` keeps WAL streaming alive and files completed segments, and `barman switch-wal` closes the current segment so it reaches R2 without waiting to fill up.

### Step 4: Create `.env.barman`

```bash
AWS_ACCESS_KEY_ID=<R2_KEY_ID>
AWS_SECRET_ACCESS_KEY=<R2_SECRET>
AWS_DEFAULT_REGION=auto
AWS_REQUEST_CHECKSUM_CALCULATION=when_required
AWS_RESPONSE_CHECKSUM_VALIDATION=when_required
```

The last two lines make newer versions of the AWS library skip checksum headers that some S3-compatible services reject. They are harmless if R2 does not need them. Run `chmod 600 .env.barman` and keep the file out of git.

### Step 5: Create `barman-main.conf`

```ini
[main]
description = "Application database"
conninfo = host=db user=barman dbname=postgres
streaming_conninfo = host=db user=streaming_barman
backup_method = postgres
streaming_archiver = on
slot_name = barman
create_slot = auto
archiver = off
retention_policy = RECOVERY WINDOW OF 7 DAYS
post_backup_retry_script = 'barman-cloud-backup --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com s3://<BUCKET>/barman ${BARMAN_SERVER}'
pre_archive_retry_script = 'barman-cloud-wal-archive --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com s3://<BUCKET>/barman ${BARMAN_SERVER}'
```

### Step 6: Start it and take the first backup

```bash
docker compose up -d --build barman

# Passwords for the two roles, stored in the sidecar's volume
docker compose exec barman bash -c \
  'printf "db:5432:*:barman:%s\ndb:5432:*:streaming_barman:%s\n" "<BARMAN_PW>" "<STREAM_PW>" > ~/.pgpass && chmod 600 ~/.pgpass'

docker compose exec barman barman cron
docker compose exec barman barman switch-wal --force --archive main
docker compose exec barman barman check main
docker compose exec barman barman backup main --wait
docker compose exec barman barman list-backups main
```

`barman check main` should report OK on every line. Then confirm the relay reached R2:

```bash
docker compose exec barman barman-cloud-backup-list \
  --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com s3://<BUCKET>/barman main
```

### Step 7: Schedule

Barman's retention policy prunes only the sidecar's copy, so the R2 copy needs its own cleanup job. Add both lines to the crontab of a host user that can run Docker.

```bash
# Base backup daily at 01:30, R2 cleanup at 02:30
30 1 * * * cd /path/to/project && docker compose exec -T barman barman backup main --wait >> $HOME/barman-cron.log 2>&1
30 2 * * * cd /path/to/project && docker compose exec -T barman barman-cloud-backup-delete --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com --retention-policy "RECOVERY WINDOW OF 7 DAYS" s3://<BUCKET>/barman main >> $HOME/barman-cron.log 2>&1
```

For a health check, `docker compose exec barman barman check main` exits non-zero when anything is wrong.

The R2 copies are not encrypted on the client side in this setup, unlike the pgBackRest one.

## Restore

Restores with Barman take more manual steps than with pgBackRest, because the sidecar writes into the Postgres volume as a different user and the stock image has no tool to fetch WAL from R2.

### Point-in-time restore from the sidecar (VPS intact)

Use this for a bad migration or an accidental delete. Pick a backup that finished before your target time from `barman list-backups main`, and always write the timezone offset in the target time.

```bash
docker compose stop db          # stop your app service too

# Move the current data aside and create an empty directory Barman can write to
docker compose exec -u root barman bash -c \
  'mv /pgdata/18/docker /pgdata/18/docker.old && install -d -o barman -g barman -m 700 /pgdata/18/docker'

docker compose exec barman barman restore main <backup_id> /pgdata/18/docker \
  --target-time "2026-11-18 13:30:00+05:30" --target-action promote

# Hand the files to the postgres user and start
docker compose exec -u root barman chown -R postgres:postgres /pgdata/18/docker
docker compose up -d db
docker compose logs -f db
```

Barman copies the WAL it needs into the restored directory, so the stock Postgres container can replay it without extra tools. Once the data checks out, delete `/pgdata/18/docker.old` and take a new base backup.

### Restore from R2 onto a new VPS

You need the Compose file, `barman-main.conf` and `.env.barman` from the old machine, so keep copies of them somewhere other than the VPS. Because only the sidecar image can fetch WAL from R2, recovery runs inside the sidecar first, and the stock `db` container starts afterwards.

```bash
docker compose up -d --build barman        # do not start db yet

# Find the backup to restore
docker compose exec barman barman-cloud-backup-list \
  --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com s3://<BUCKET>/barman main

# Fetch the base backup into the Postgres volume
docker compose exec -u root barman install -d -o postgres -g postgres -m 700 /pgdata/18/docker
docker compose exec -u postgres barman barman-cloud-restore \
  --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com s3://<BUCKET>/barman main <backup_id> /pgdata/18/docker
```

Then open a shell in the sidecar as the `postgres` user (`docker compose exec -u postgres barman bash`) and run the recovery there:

```bash
cat >> /pgdata/18/docker/postgresql.auto.conf <<'EOF'
restore_command = 'barman-cloud-wal-restore --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com s3://<BUCKET>/barman main %f %p'
recovery_target_action = 'promote'
EOF
touch /pgdata/18/docker/recovery.signal

pg_ctl -D /pgdata/18/docker -w -t 3600 start

# Wait until WAL replay has finished and the server has promoted itself
until [ "$(psql -U <superuser> -d postgres -Atc 'select pg_is_in_recovery()')" = "f" ]; do sleep 2; done

psql -U <superuser> -d postgres -c "ALTER SYSTEM RESET restore_command;"
pg_ctl -D /pgdata/18/docker -w stop
exit
```

For a point in time rather than the latest state, add a `recovery_target_time = '2026-11-18 13:30:00+05:30'` line next to `restore_command`.

Do not skip the wait loop. If Postgres is stopped before it promotes, the stock container would later fail to fetch the remaining WAL and could come up with data missing.

Finally start the real service and re-seed the sidecar:

```bash
docker compose up -d db
# recreate ~/.pgpass in the sidecar as in setup step 6, then:
docker compose exec barman barman cron
docker compose exec barman barman backup main --wait
```
