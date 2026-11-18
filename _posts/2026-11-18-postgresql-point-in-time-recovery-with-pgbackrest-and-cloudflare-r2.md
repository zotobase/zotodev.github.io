---
title: "PostgreSQL Point-in-Time Recovery with pgBackRest, Docker Compose, and Cloudflare R2"
---

A daily `pg_dump` can lose up to 24 hours of data. Point-in-time recovery (PITR) closes that gap: a periodic base backup plus continuous WAL shipping lets you restore a self-hosted PostgreSQL to any minute inside the retention window.

This guide sets up PITR with [pgBackRest](https://pgbackrest.org/) for PostgreSQL 18 running in Docker Compose on a single VPS, with Cloudflare R2 as the backup repository. If you would rather keep the stock `postgres:18` image untouched, see the [Barman sidecar alternative](/posts/postgresql-point-in-time-recovery-with-barman-and-cloudflare-r2/).

Treat the restore drill near the end as the acceptance test. A backup setup is only proven once a restore from it has worked on your own stack.

## Before you start

### Assumptions

- The Compose service is called `db` and runs `postgres:18`.
- The data volume is mounted at `/var/lib/postgresql`, the default for the 18 image, so the data directory is `/var/lib/postgresql/18/docker` ([Docker Hub](https://hub.docker.com/_/postgres)). Confirm with `docker compose exec db psql -U <superuser> -c "show data_directory"`.
- The database port is not published to the internet.
- Commands run from the Compose project directory on the VPS.

### Placeholders

| Placeholder | Meaning |
| --- | --- |
| `<superuser>` | The value of `POSTGRES_USER` in your Compose file (`postgres` if you never set it) |
| `<ACCOUNT_ID>` | Your Cloudflare account ID, as shown in the R2 S3 API endpoint |
| `<BUCKET>` | The R2 bucket that holds the backups |
| `<R2_KEY_ID>` and `<R2_SECRET>` | Access Key ID and Secret Access Key of the R2 API token |

### Prepare R2

1. Create a bucket used only for these backups, for example `pg-backups`.
2. Create an R2 API token with Object Read & Write permission scoped to that bucket, and note the Access Key ID and Secret Access Key.
3. Do not add a lifecycle rule that deletes objects in this bucket. pgBackRest manages retention, and deleting WAL files underneath it breaks restores.

Keep an existing daily `pg_dump` running as an independent second backup.

## Why a custom image is needed

Postgres itself runs the `archive_command` for every WAL segment, inside its own container. The backup tool's binary therefore has to exist in that container, and the official image does not include one. The build below is the official image plus one package, defined inline in the Compose file, so no separate Dockerfile or registry is needed.

## Setup

The setup is five steps and one short restart of Postgres, because `archive_mode` only takes effect at startup.

### Step 1: Add pgBackRest to the image and turn on archiving

Replace the `image: postgres:18` line of your `db` service with the `build` and `image` lines below, and add the `env_file`, the config mount and the `command`. Everything else in the service stays as it is.

```yaml
services:
  db:
    build:
      dockerfile_inline: |
        FROM postgres:18
        RUN apt-get update \
         && apt-get install -y --no-install-recommends pgbackrest ca-certificates \
         && rm -rf /var/lib/apt/lists/*
    image: local/postgres-pgbackrest:18
    env_file:
      - .env.pgbackrest
    volumes:
      - pgdata:/var/lib/postgresql        # your existing data volume, unchanged
      - ./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro
    command:
      - postgres
      - -c
      - archive_mode=on
      - -c
      - archive_command=pgbackrest --stanza=main archive-push %p
      - -c
      - archive_timeout=60
```

If the service already has a `command`, append the three `-c` settings to it. `archive_timeout=60` makes Postgres ship a WAL segment at least once a minute while there is write activity, which caps data loss at about a minute.

### Step 2: Create `pgbackrest.conf`

Put this next to the Compose file. It holds no secrets, so `chmod 644 pgbackrest.conf` is fine, and the container's `postgres` user must be able to read it.

```ini
[global]
repo1-type=s3
repo1-s3-endpoint=<ACCOUNT_ID>.r2.cloudflarestorage.com
repo1-s3-bucket=<BUCKET>
repo1-s3-region=auto
repo1-s3-uri-style=path
repo1-path=/pgbackrest
repo1-retention-full=2
repo1-cipher-type=aes-256-cbc
start-fast=y
log-level-console=warn
log-level-file=off

[main]
pg1-path=/var/lib/postgresql/18/docker
pg1-user=<superuser>
```

`repo1-cipher-type` encrypts everything before it leaves the VPS. If you do not want client-side encryption, remove that line and the cipher passphrase in step 3. Decide now, because it cannot be changed after the repository is created.

### Step 3: Create `.env.pgbackrest`

pgBackRest reads any option from a `PGBACKREST_` environment variable, which keeps the secrets out of the config file.

```bash
PGBACKREST_REPO1_S3_KEY=<R2_KEY_ID>
PGBACKREST_REPO1_S3_KEY_SECRET=<R2_SECRET>
PGBACKREST_REPO1_CIPHER_PASS=<output of: openssl rand -base64 48>
```

Run `chmod 600 .env.pgbackrest` and keep it out of git. Store the cipher passphrase in your password manager as well: without it the backups cannot be read.

### Step 4: Rebuild and restart

```bash
docker compose build --pull db
docker compose up -d db
```

Postgres will log archive failures until step 5 creates the repository. That is expected, and the queued WAL is shipped as soon as it exists.

### Step 5: Create the repository, verify, take the first backup

```bash
docker compose exec -u postgres db pgbackrest --stanza=main --log-level-console=info stanza-create
docker compose exec -u postgres db pgbackrest --stanza=main --log-level-console=info check
docker compose exec -u postgres db pgbackrest --stanza=main --log-level-console=info --type=full backup
docker compose exec -u postgres db pgbackrest --stanza=main info
```

`check` forces a WAL segment through the archive and confirms it arrived in R2. If it passes, archiving works. `info` should then list one full backup and a WAL range.

Future image upgrades are the same two commands as step 4.

## Schedule and health

A weekly full backup plus daily differentials gives a restore window of 7 to 14 days with the retention set in step 2. WAL shipping runs by itself; only the base backups need a schedule.

### Schedule

Add these to the crontab of a host user that can run Docker (`crontab -e`). Times are in the server's timezone.

```bash
# Full backup on Sunday, differential on the other days, at 01:30
30 1 * * 0   cd /path/to/project && docker compose exec -T -u postgres db pgbackrest --stanza=main --type=full backup >> $HOME/pgbackrest-cron.log 2>&1
30 1 * * 1-6 cd /path/to/project && docker compose exec -T -u postgres db pgbackrest --stanza=main --type=diff backup >> $HOME/pgbackrest-cron.log 2>&1
```

### Retention

`repo1-retention-full=2` keeps the two newest full backups, their differentials, and all WAL needed to restore from them. Older backups and WAL are removed automatically after each successful backup. Raise the number for a longer window.

### Health checks

```bash
# Backups and the WAL range held in R2
docker compose exec -u postgres db pgbackrest --stanza=main info

# Is WAL shipping current?
docker compose exec db psql -U <superuser> -d postgres -c \
  "select last_archived_time, failed_count, last_failed_time from pg_stat_archiver;"
```

`last_archived_time` should be recent whenever the database has had writes, and `last_failed_time` should be empty or older than it.

The one failure to watch for is archiving that stays broken, for example after the R2 token is revoked. Postgres then keeps every WAL file in the data volume until the disk fills. Run the query above weekly, or add it to whatever monitoring you already have.

## Restore

Every restore runs in a one-off container from the same image, because Postgres must be stopped while its data directory is rewritten. Do the drill once right after setup, and again every few months.

Two rules for the target time: it must be later than the end of your oldest base backup, and at least two minutes in the past, because WAL reaches R2 up to a minute late. Always write the timezone offset.

### Restore drill (does not touch live data)

This restores into a scratch directory inside a throwaway container and starts a second Postgres there on port 5433. Change a row, note the time just before the change, wait two minutes, then restore to that time.

```bash
docker compose run --rm --name pitr-drill -u postgres db bash -c '
  mkdir -m 700 /tmp/restore &&
  pgbackrest --stanza=main --pg1-path=/tmp/restore --log-level-console=info \
    --type=time "--target=2026-11-18 13:30:00+05:30" --target-action=promote \
    --archive-mode=off restore &&
  postgres -D /tmp/restore -p 5433 -c archive_mode=off'
```

`--archive-mode=off` stops the scratch copy from pushing its own WAL into the real repository. When the log says the database is ready to accept connections, check the data from a second terminal:

```bash
docker exec -it pitr-drill psql -p 5433 -U <superuser> -d <your_database> -c "select ... ;"
```

Press Ctrl+C in the first terminal to finish. The container and the scratch copy are removed.

### Real point-in-time restore (replaces live data)

1. If the database still runs, take a `pg_dump` first so you can go back to the current state.
2. Stop the application and the database: `docker compose stop db` (plus your app service).
3. Restore. `--delta` rewrites only the files that differ, so it is quick.

   ```bash
   docker compose run --rm -u postgres db \
     pgbackrest --stanza=main --log-level-console=info --delta \
     --type=time "--target=2026-11-18 13:30:00+05:30" --target-action=promote restore
   ```

4. Start it and watch the replay: `docker compose up -d db && docker compose logs -f db`. Wait for the ready message, then check the data and start the app.
5. Take a new full backup with the command from setup step 5.

To restore to the latest available state instead of a point in time, leave out the `--type`, `--target` and `--target-action` options.

### Restore onto a new VPS

You need three files from the old machine: the Compose file, `pgbackrest.conf` and `.env.pgbackrest`. Keep copies of them somewhere other than the VPS, or the backups in R2 cannot be used.

```bash
docker compose build db
# Do not start db yet: an empty volume would be initialised as a new, empty cluster.
docker compose run --rm -u postgres db bash -c \
  'mkdir -p -m 700 "$PGDATA" && pgbackrest --stanza=main --log-level-console=info restore'
docker compose up -d db
```

This assumes a named volume. With a bind-mounted host directory, make sure it is owned by UID 999, the container's `postgres` user, before the restore.
