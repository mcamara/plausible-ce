# AGENTS.md

This repo deploys Plausible Community Edition (CE) to https://plausible.farnit.com with Kamal. It is a fork of [plausible/community-edition](https://github.com/plausible/community-edition). The `kamal-deploy` branch holds the deploy setup. Upstream's `compose.yml` and `README.md` stay for reference. Production does not use Docker Compose.

## Files

| Path | Purpose |
|---|---|
| `Dockerfile` | One line: `FROM ghcr.io/plausible/community-edition:<version>`. Kamal deploys only images that it builds, so this file wraps the upstream image. Its tag is the Plausible version in production. |
| `config/deploy.yml` | The Kamal config for the app and both databases. It matches the containers on the server. |
| `.kamal/secrets` | Reads the secrets from 1Password. It holds references, not values. |
| `bin/upgrade-prepare`, `bin/upgrade-cutover`, `bin/upgrade-rollback` | The upgrade procedure. See "Upgrade Plausible". |
| `clickhouse/*.xml` | Upstream ClickHouse config. Production mounts only `logs.xml` and `ipv4-only.xml`. |
| `compose.yml`, `README.md` | Upstream files. Use them to see what changed between versions. |

## Production

| Item | Value |
|---|---|
| Server | `116.203.58.6` (Hetzner, hostname `api-contacts-staging`). Other Kamal apps share the server and its kamal-proxy. |
| SSH user | `plausible`. All Plausible data is under `/home/plausible`. |
| App | Kamal service `plausible`. Container `plausible-web-<git sha>` listens on port 8000. kamal-proxy serves `plausible.farnit.com` with TLS and checks `/api/health`. |
| Images | Kamal's local registry, `localhost:5555/plausible`. The server pulls through an SSH tunnel from the machine that deploys. |
| Postgres | Accessory `db`, container `plausible-db`. Image `postgres:18`, port 5433, user `app_user`, database `plausible_db`. Data in `/home/plausible/plausible-db/data`. |
| ClickHouse | Accessory `events-db`, container `plausible-events-db`. Image `clickhouse/clickhouse-server:25.12-alpine`, port 8123, user `app_user` with a password, database `plausible_events_db`. Data in `/home/plausible/plausible-events-db/event-data`. |
| Secrets | 1Password item `Plausible` in vault `Contacts`. `.kamal/secrets` builds `DATABASE_URL` and `CLICKHOUSE_DATABASE_URL` from the passwords. |

Production does not match upstream's `compose.yml`. Postgres and ClickHouse run newer images, with a user and a password. Do not copy database settings from `compose.yml` into `config/deploy.yml`. Copy only the changes that a release note requires.

## Rules

- Do not rename the service or the accessories. Kamal derives container names and hostnames from them.
- Do not change the accessory `directories`. A wrong data path boots an empty database.
- Do not change `SECRET_KEY_BASE`. Plausible derives the 2FA encryption key from it. A new value logs out every user and breaks 2FA.
- Do not commit secret values. This repo is public.
- Do not run `kamal accessory reboot` without a backup. The command removes the container and starts a new one from the config.
- Do not downgrade ClickHouse. An older ClickHouse may not read data that a newer one wrote.
- Commit changes before a deploy. Kamal builds from the committed HEAD, not from the working tree.

## How AI agents work here

- Agents cannot read `.kamal/secrets`. A deny rule in Marc's settings blocks it.
- Agent permissions block commands against the production server, such as `ssh` and `kamal`. Put those commands in a script. Marc runs the script with `! bash <script>`, and the agent reads the output.
- Never print a secret value. To compare a secret with production, compare a hash prefix (`sha256sum | cut -c1-12`).

## Upgrade Plausible

1. Find the latest release: `gh release list -R plausible/analytics --limit 10`.
2. Find the version in production: `grep '^FROM' Dockerfile`.
3. Read the release notes of every version between the two: `gh release view <tag> -R plausible/analytics`. Look for these items:
   - An "Upgrade" section with more steps than a new image tag.
   - A new ClickHouse or Postgres version.
   - New or changed environment variables.
   - Changed ClickHouse config files.
   - Security fixes. Go directly to a fixed version. Do not deploy a version that the notes call vulnerable.
4. Fetch the upstream branch of the target version: `git fetch upstream refs/heads/<tag>:refs/remotes/upstream/<tag>`.
5. Compare the upstream files: `git diff HEAD upstream/<tag> -- compose.yml clickhouse/`.
6. Merge the upstream branch: `git merge upstream/<tag>`.
7. Set the new tag in `Dockerfile`.
8. If step 3 or step 5 requires a change to a database, stop. Plan that change on its own. See "Change a database".
9. Run `kamal config` to validate the config.
10. Commit and push.
11. Marc runs `bin/upgrade-prepare`. It checks the secrets, builds the image, and copies the image to the server. The site stays up.
12. Marc runs `bin/upgrade-cutover`. The site is down for a few minutes. The script stops the app, makes the backups, deploys, and prints the row counts before and after.
13. Check the output. The health check must report `ok` for every item. No row count can be lower than before.
14. Ask Marc to open the Plausible dashboard and to check that new visits appear in the realtime view.
15. If the cutover fails, run the command that the script prints. After a failed deploy step, that command is `bin/upgrade-rollback <stamp> <old version>`.
16. After a few days without problems, Marc deletes the server backup: `ssh plausible@116.203.58.6 rm -r backups/<stamp>`. The Postgres dump stays in `~/backups/plausible/` on the laptop.
17. Add a line to "History".

## Change a database

The upgrade scripts do not change the databases. A Postgres or ClickHouse change needs its own plan.

- **Postgres major version:** dump the database, boot the new image on a new data directory, and restore the dump. Keep the old directory until the new version runs without problems. The Postgres 18 image keeps its data in `/var/lib/postgresql/18/docker`, so the mount target is `/var/lib/postgresql`.
- **ClickHouse:** copy the data folder while ClickHouse is stopped (`bin/upgrade-cutover` shows how), change the image in `config/deploy.yml`, and run `kamal accessory reboot events-db`.

## Check production

Marc runs these commands. They change nothing.

- `kamal app details` shows the running app container.
- `kamal app logs` shows the app logs.
- `curl https://plausible.farnit.com/api/health` reports the state of Postgres, ClickHouse, and the caches.

## History

- 2025-12-29: First deploy with Plausible v2.1.4, based on [a Kamal guide](https://kyrylo.org/kamal/2024/11/15/how-to-self-host-plausible-analytics-with-kamal.html). The running setup differs from the guide, and its repo was never pushed.
- 2026-10-25: `config/deploy.yml` rebuilt from the containers on the server. Plausible upgraded from v2.1.4 to v3.2.1. Postgres 18 and ClickHouse 25.12 unchanged.
