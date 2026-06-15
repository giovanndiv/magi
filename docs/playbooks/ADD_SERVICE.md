# Playbook: Add a New Service to the MAGI Stack

> This file is a spec written to **you, Claude Code**, as the executor. Read
> `CLAUDE.md` first, then follow these steps in order to add a single new
> service to the MAGI Docker Compose stack. Stop and escalate to the human at
> every point marked **STOP**.

## Purpose

This playbook lets you add one new service to the MAGI Docker Compose stack end
to end: cut a branch, research the upstream, edit `docker-compose.yml`, deploy
and test on `nerv` over SSH, troubleshoot iteratively from container logs, open
a PR with the automated `@claude` review, and merge once the review is clean —
while stopping to escalate to the human at the defined checkpoints (missing
input, broken preconditions, required server-side secrets, exhausted
troubleshooting or review rounds). Operate within the new service's scope only;
never touch unrelated services, storage, or the VPN.

## Input Contract

The human provides only two things:

1. **The service name** — used for the container name, branch, subdomain, and
   config path.
2. **The upstream source** — either a Docker image reference (e.g.
   `ghcr.io/org/app:latest`) or a GitHub repo URL for the service.

Everything else is derived from the conventions already in the repo. If either
input is missing or ambiguous (e.g. no image and no repo, or a name that
collides with an existing service), **STOP and ask** before doing anything.

## Preconditions (verify before starting)

Run these checks first. If any fails, **STOP** with the stated message — do not
proceed.

- **Passwordless SSH to nerv works.** Test:
  `ssh gendo@nerv "echo connected"`
  If it prompts for a password or fails, **STOP** and tell the human to set up
  the SSH key before continuing.
- **Working tree is clean, on `master`, up to date with origin.** Test:
  `git status` (clean), `git branch --show-current` (`master`),
  `git fetch && git status` (up to date). If not, **STOP** and ask how to
  proceed.
- **The server config is currently valid.** Test:
  `ssh gendo@nerv "cd ~/magi && docker compose config"`
  If it fails, **STOP** — a broken config breaks variable resolution globally
  and must be fixed before any new service is added.

## Build Steps

1. **Branch.** Check out a new branch named `feat/add-<service>`.

2. **Research the upstream — do not guess.** Fetch the upstream repo's compose
   file / docs, or the image's documented environment variables, ports, and
   volumes from the official source (Docker Hub, GHCR, or the project's GitHub
   README). Confirm the documented env vars, the web UI port, and any required
   volume paths before writing anything.

3. **Add the service block to `docker-compose.yml`**, matching the conventions
   of the existing services in that file:
   - `container_name: <service>`
   - `environment` with `PUID=${USER_ID}` / `PGID=${GROUP_ID}` (or the image's
     documented equivalents) and `TZ=${TIMEZONE}`
   - config volume under `${CONFIG_ROOT:-.}/<service>:/config`
   - data volume under `${DATA_ROOT}` **only if** the service needs media access
   - `networks: [nerv]`
   - `restart: always`

4. **If the service is web-exposed, add Traefik labels** following the existing
   subdomain pattern (see any *arr service for reference):
   - ``traefik.http.routers.<service>.rule=Host(`${<SERVICE>_HOSTNAME}`)``
   - `traefik.http.routers.<service>.tls=true`
   - `traefik.http.routers.<service>.tls.certresolver=myresolver`
   - `traefik.http.services.<service>.loadbalancer.server.port=<webui-port>`
   - Add a matching `<SERVICE>_HOSTNAME=<service>.${BASE_HOSTNAME}` entry to
     `.env.example`.

5. **Gate non-core services behind a compose profile.** If the service is not
   essential to the media pipeline, add `profiles: [<service>]` so it does not
   start by default, rather than letting it come up with the core stack.

6. **Add a homepage label block** matching the format other services use:
   `homepage.group`, `homepage.name`, `homepage.icon`, `homepage.href`, and
   `homepage.widget.*` if the service has a supported widget.

7. **Add any new env vars to `.env.example`** with safe placeholder values.
   **NEVER** write real secrets to any tracked file.

8. **Add a healthcheck using `wget`** (not `curl` — it is absent from many
   minimal images). Include `timeout` and `start_period`. Use `cleanuparr` as
   the reference pattern. **If the upstream image already ships its own
   healthcheck, do not override it.**

9. **Validate locally before pushing.** Copy `.env.example` to `.env` and run
   `docker compose config`. It must succeed (and resolve the new service)
   before you push.

## Deploy and Test Loop (on nerv over SSH)

1. Commit and push the `feat/add-<service>` branch.

2. SSH to nerv and sync the branch:
   `ssh gendo@nerv "cd ~/magi && git fetch && git checkout feat/add-<service> && git pull"`

3. **If new env vars are required for the service to run**, tell the human the
   exact keys to add to the real `~/magi/.env` on the server, and **wait** — do
   not invent values. Real secrets live only on the server, never in git.

   - **Exception — non-secret, derived values** (e.g. `<SERVICE>_HOSTNAME`,
     which is always `<service>.${BASE_HOSTNAME}`): these are deterministic and
     identical to the `.env.example` entry, so you may append them to the server
     `.env` yourself over SSH rather than waiting. Quote literally so compose
     expands `${BASE_HOSTNAME}` (do **not** let your shell expand it):
     `ssh gendo@nerv "cd ~/magi && grep -q '^<SERVICE>_HOSTNAME=' .env || echo '<SERVICE>_HOSTNAME=<service>.\${BASE_HOSTNAME}' >> .env"`

4. **If the service needs a server-side config/credentials file in its bind-
   mounted data dir** (e.g. an auth `users.yml`): note that when the container
   first starts, Docker creates the host-side bind-mount dir
   (`${CONFIG_ROOT:-.}/<service>`) owned by **root**, so `gendo` cannot write
   into it over the (non-`sudo`) SSH session. `gendo` also does **not** have
   passwordless `sudo`. Therefore:
   - Ask the human to run, in a real terminal with a TTY (the in-session `!`
     prefix has no TTY, so `sudo` cannot prompt):
     `ssh -t gendo@nerv "sudo chown gendo:gendo ~/magi/<service>"`
   - After the human confirms, you can write the file over plain SSH.
   - Generate hashed credentials with the image's own tool and redirect the
     output to the server-side file only — **never** echo a real secret into a
     tracked file, and add the data dir to `.gitignore` so it can never be
     committed.

5. Bring the service up:
   `docker compose up -d <service>`
   (use `--force-recreate` if env vars used in **labels** changed).

6. **Determine health:**
   - **If the container defines a healthcheck:** poll
     `docker inspect --format '{{.State.Health.Status}}' <service>`
     until it reads `healthy` or `unhealthy`, up to a **3-minute** timeout.
   - **If the container has NO healthcheck:** confirm via `docker ps -a` that
     the service status is **not** `Restarting` and has been stable for at least
     30 seconds, then scan recent logs for an obvious readiness line or a fatal
     error.

7. **Troubleshooting loop (max 5 iterations):**
   - On unhealthy / restarting / error: pull logs with
     `docker compose logs <service> --since 5m`, read them, and form a
     hypothesis.
   - Apply a fix **only** to the new service's own block in
     `docker-compose.yml` (or instruct the human to adjust a server-side env var
     if that's the root cause). Commit, push, re-pull on the server, run
     `docker compose up -d --force-recreate <service>`, and re-check health.
   - Increment the iteration counter. **After 5 failed iterations, STOP** and
     report the logs and everything you tried to the human. Do not continue
     blindly.

## PR and Review Loop

1. Once the service is healthy, ensure all changes are committed and pushed.

2. Open the PR:
   `gh pr create --base master --head feat/add-<service> --title "feat: add <service>"`

3. Trigger the review:
   `gh pr comment --body "@claude review"`

4. Poll until the GitHub Claude bot finishes:

       while true; do
         COMMENT=$(gh pr view --json comments --jq '.comments[-1].body')
         if echo "$COMMENT" | grep -q "Claude finished"; then
           echo "Review complete:"; echo "$COMMENT"; break
         fi
         sleep 30
       done

5. Read the review. If there are high-level / blocking warnings, resolve them
   (again, **only** touching the new service's scope unless the human approves
   wider changes), push, and re-run the review + poll **once** more.

6. **After two review rounds:**
   - If the review is clean, proceed to **Merge**.
   - If warnings remain after the second round, **STOP** and escalate to the
     human with a summary. Do not loop a third time.

## Merge

On a clean review:

1. `gh pr merge --merge`
2. `git checkout master && git pull origin master`
3. Delete the local and remote branch:
   `git branch -d feat/add-<service> && git push origin --delete feat/add-<service>`
4. On nerv: `ssh gendo@nerv "cd ~/magi && git pull origin master"`
5. Confirm the service is still healthy after pulling on `master`.

## Guardrails (must never do without stopping to ask)

- **Never modify any service block other than the new one.**
- **Never modify the storage mounts, the mergerfs/snapraid data layout, or the
  qBittorrent / VPN (`network_mode: "service:vpn"`) configuration.**
- **Never write real secrets, API keys, passwords, IPs, or private tracker
  details to any tracked file.**
- **Never disable or weaken another service's healthcheck.**
- **Never force-merge past unresolved high-level review warnings.**
- **Never exceed 5 troubleshooting iterations or 2 review rounds without
  escalating.**
- **If `docker compose config` ever fails, fix that before anything else** — a
  broken config breaks variable resolution for the whole stack.
