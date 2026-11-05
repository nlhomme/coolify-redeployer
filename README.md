# coolify-redeployer

A Coolify service that redeploys other Coolify resources on a cron schedule (`CRON_SCHEDULE`, default every day at midnight Europe/Paris), through the Coolify API. It runs a BusyBox `crond` in an `alpine:3.24` container.
All API calls stay inside your instance (`http://coolify:8080`). The only outbound traffic is the Alpine package download (`curl`, `tzdata`) at each container start.

## Why ?

The apps on my Coolify instance use images that get updated upstream. Picking up a new image means redeploying the app, and doing that by hand for every app is a chore I will forget.

This service does it for me: on a schedule it asks Coolify to redeploy a list of resources, using Coolify's own API. That gives:

- **Nothing to remember**: the redeploys happen at night without me.
- **No extra moving parts**: one small container, `alpine` plus `curl`, `tzdata` and `crond`, with no custom image and no external service. The calls never leave the instance.
- **Easy to change**: the resources, the schedule and the token are env variables, so nothing is edited in the compose file after setup.
- **Visible**: every run prints one line per resource in the Runtime Logs, and Coolify marks the service unhealthy when the next run would fail.

It is not a rollout tool: it queues a normal deployment for each resource and does not wait for the result or check that the app came back up.

## Setup

1. In Coolify **Settings** (Advanced), enable **API Access** (it is off by default; if you restrict **Allowed IPs**, include the Docker network the container runs in). Then create an API token (**Keys & Tokens → API tokens**) with the **deploy** and **read** abilities. The job needs `deploy`; the healthcheck needs `read` (for `GET /api/v1/version`), which also confirms the token is valid. `write` and `delete` are not needed.
2. In Coolify, create a service from `docker-compose.yml`.
3. In the service **Environment Variables**, set all required variables (the container refuses to start without them):
   - `TOKEN`: the bare token from step 1, such as `2|abc...`, with no quotes (mark it secret, lock it).
   - `UUIDS`: the UUIDs of the resources to redeploy, separated by spaces (e.g. `uuid1 uuid2 uuid3`). Coolify UUIDs are lowercase letters and digits; the container refuses any other character. Changing the list later only needs an env edit and a restart, not a compose edit.
   - `CRON_SCHEDULE` (optional): a 5-field cron expression in the `TZ` timezone (Europe/Paris). Defaults to `0 0 * * *` (every day at midnight). Changing it needs a restart.
4. In the service settings, enable **Connect to Predefined Network**. Without it the container cannot resolve `coolify:8080` and every call fails with `HTTP 000`.
5. Deploy the service.

## How it works

At startup the container:

1. checks `TOKEN`, `UUIDS` and `CRON_SCHEDULE`, and exits if one is invalid (see below);
2. installs `curl` and `tzdata` (`tzdata` makes `TZ` effective). If the install fails, `set -e` exits and `restart: unless-stopped` retries;
3. writes the token and UUIDs to `/etc/redeploy.token` and `/etc/redeploy.uuids` (mode 600), since cron jobs do not inherit the container environment;
4. writes `/tmp/redeploy.sh`: one `POST /api/v1/deploy` with `{"uuid": ..., "force": false}` per UUID, 10s connect / 60s total timeout so a stalled API cannot hang the job;
5. installs the crontab `$CRON_SCHEDULE` and `exec`s `crond` as PID 1, so it stops cleanly on SIGTERM.

| Startup error | Cause |
|---|---|
| `TOKEN and UUIDS are required` | one of them is empty |
| `UUIDS must be space-separated UUIDs (letters/digits only)` | comma, quote, newline, `*`… in `UUIDS`. Caught at startup instead of at the next run, and keeps each UUID safe to put in the JSON body |
| `CRON_SCHEDULE must have 5 fields, got N (check it on https://crontab.guru)` | wrong field count. With a 6-field (seconds-first) expression, crond takes the 6th field as the start of the command (e.g. `* /tmp/redeploy.sh`), so the job never runs and nothing marks the container unhealthy. Out-of-range values (`0 25 * * *`) are not caught: check on [crontab.guru](https://crontab.guru) |

## Health

The healthcheck calls `GET /api/v1/version` every minute with the same token as the job. Coolify shows the service as **unhealthy** when curl is missing, `coolify:8080` is unreachable, or the token is rejected, which are the cases where the next scheduled run would fail. It is also briefly unhealthy while Coolify itself restarts or upgrades. To see why it is unhealthy:

```
docker inspect cronjob-<service-uuid> --format '{{json .State.Health}}'
```

## Logs

The job output goes to `/proc/1/fd/1` and `crond` logs to `/dev/stdout` (`-L`), so everything appears in the Coolify **Runtime Logs** of the `cronjob` container. At the default level (`-l 8`) you see the startup lines, one launch line per scheduled run, and the script's `OK:`/`FAILED:` lines.

The job prints one line per UUID, ending with Coolify's raw JSON response (first 300 bytes):

```
OK: deployment queued for <uuid>: {"deployments":[{"message":"Application my-app deployment queued.",...}]}
FAILED: deployment could not be queued for <uuid> (HTTP 401): {"message":"Unauthenticated."}
```

Check the message, not just `OK:`: a 200 can also mean nothing new was queued (`Deployment already queued for this commit.` or `Unauthorized to deploy this application.`). See Pitfalls for what each HTTP code means.

Run the job manually (it queues real deployments). Coolify renames the container to `cronjob-<service uuid>`:

```
docker exec cronjob-<service-uuid> /tmp/redeploy.sh
```

## Pitfalls (all hit while building this)

- **Escape `$` as `$$`** in the compose `command:`. Compose interpolates `$VAR` before the shell runs, which turns variables blank. The only single `$` are `${TOKEN}`, `${UUIDS}` and `${CRON_SCHEDULE:-0 0 * * *}` under `environment:`, where substitution is wanted.
- **Never put the token in the crontab line.** `crond` logs the full command at level 8, which leaks the token. Coolify tokens contain `|`, which the shell also reads as a pipe. The script reads the token from `/etc/redeploy.token` instead.
- **`-l` is verbosity, lower means more verbose.** `-l 2` prints debug output (schedule bitmaps, per-minute wakeups). Use `-l 8`.
- **`container_name` is ignored by Coolify**, which names the container `cronjob-<service uuid>`.
- **`HTTP 000`** means the container cannot reach `coolify:8080` (see Setup step 4), or the API did not answer within the timeout. **`HTTP 401`** (`Unauthenticated.`) means the token is wrong or revoked. **`HTTP 403`** means the token lacks the `deploy` ability (`Missing required permissions: deploy`), API Access is disabled (`API is disabled.`), or the container's IP is not in **Allowed IPs** (`You are not allowed to access the API.`, see Setup step 1); the logged body tells which. **`HTTP 404`** (`No resources found.`) means the UUID does not exist in the token's team.

## License

[MIT](LICENSE)
