# Playbook: Add a New Service to the MAGI Stack

> This file is a spec written to **you, Claude Code**, as the executor. Read
> `CLAUDE.md` first, then follow these steps in order to add a single new
> service to the MAGI Docker Compose stack. Stop and escalate to the human at
> every point marked **STOP**.

> **Note:** if the service's value lives in a hand-authored config file
> containing secrets, external API calls, or destructive-action settings (e.g.
> cross-seed), use `CONFIGURE_SERVICE.md` instead — either standalone or as a
> second pass after this playbook deploys the compose block.

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
   - **no `networks:` key** — services rely on the compose default network, which
     is defined once at the bottom of the file as `default: { name: nerv }`.
     Adding an explicit `networks:` list diverges from every existing service.
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
   start by default, rather than letting it come up with the core stack. Note
   that a profiled service will only run once its profile is listed in
   `COMPOSE_PROFILES` in the server `.env` (see the Deploy loop) — gating it does
   not start it.

6. **Add a homepage label block** matching the format other services use:
   `homepage.group`, `homepage.name`, `homepage.icon`, `homepage.href`, and
   `homepage.widget.*` if the service has a supported widget.

7. **Add any new env vars to `.env.example`** with safe placeholder values.
   **NEVER** write real secrets to any tracked file. If the service stores
   credentials or config in a bind-mounted dir (e.g. an auth `users.yml` under
   `${CONFIG_ROOT:-.}/<service>`), add that dir to `.gitignore` now — matching
   the existing `/sonarr/`, `/cleanuparr/`, `/dozzle/` entries — so server-side
   secrets can never be committed.

8. **Add a healthcheck — pick the method that fits the image.** Always include
   `timeout` and `start_period`.
   - **Image already ships a healthcheck** (its Dockerfile defines `HEALTHCHECK`,
     or upstream docs show one): use it, do **not** override.
   - **Minimal / distroless image with no shell or `wget`** (single static
     binary — e.g. Dozzle): use the image's own healthcheck subcommand if it has
     one, e.g. `test: ["CMD", "/<binary>", "healthcheck"]`, or a shell-free TCP
     check. Do **not** force a `wget` test — it fails because the binary isn't in
     the image. Confirm from the upstream docs which applies.
   - **Standard image with a shell:** use `wget` (not `curl` — absent from many
     minimal images), e.g.
     `test: ["CMD", "wget", "-q", "--spider", "http://localhost:<port>/<path>"]`.
     Use `cleanuparr` as the reference pattern.

9. **Validate before pushing.**
   - **YAML sanity (always):** parse the file so a typo can't reach the server,
     and confirm the new service resolves —
     `python3 -c "import yaml; print('dozzle' in yaml.safe_load(open('docker-compose.yml'))['services'])"`
     (substitute the service name).
   - **`docker compose config` (only if Docker is available locally):** copy
     `.env.example` to `.env` and run
     `COMPOSE_PROFILES=<service> docker compose config`. The dev machine may have
     **no Docker daemon** (e.g. WSL) — if so, skip this and rely on the
     authoritative `docker compose config` run on nerv in the Deploy loop.

## Deploy and Test Loop (on nerv over SSH)

1. Commit and push the `feat/add-<service>` branch.

2. SSH to nerv and sync the branch, then run the **authoritative**
   `docker compose config` there (this is the real validation if Docker wasn't
   available on the dev machine):
   `ssh gendo@nerv "cd ~/magi && git fetch && git checkout feat/add-<service> && git pull && COMPOSE_PROFILES=<service> docker compose config >/dev/null && echo CONFIG_OK"`
   If it errors, fix the compose block before going further.

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

   - **Credential persistence checkpoint.** If this run generates or sets any
     credential a human will need later (an admin password, an API token, an auth
     secret), that credential **must be persisted to the Vaultwarden service
     vault before the run is allowed to complete**. Do not strand a secret. Run
     the following over SSH on nerv — it writes to a **dedicated Vaultwarden
     service account** whose vault holds **only** machine-generated service
     credentials, never the human's personal vault:
     - **Confirm prerequisites:** `command -v bw` succeeds and the Vaultwarden
       service is reachable (`docker compose ps vaultwarden` shows healthy). If
       either fails, **STOP** — hand the human the credential and the suggested
       entry name, and wait for explicit confirmation it has been stored before
       proceeding.
     - **Unlock fresh** (`BW_SESSION` does not persist across shells, so each
       invocation unlocks, acts, and locks):
       `export BW_SESSION=$(bw unlock --passwordfile ~/.config/bw-service-pass --raw)`
     - **Create the entry** using the naming convention `magi/<service>`, with the
       username, the generated password, and a note containing the service URL and
       creation date — build it with `bw get template item`, populate fields with
       `jq`, then `bw encode | bw create item`.
     - **Verify the write by reading it back:** `bw list items --search
       magi/<service>` must return the entry. A silent failed write is the exact
       failure this checkpoint exists to prevent — do not assume success.
     - **Lock when done:** `bw lock`.
     - **Never** echo the plaintext credential into a tracked file, a log, a
       commit message, or a PR comment.

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

8. **Smoke-test routing and auth for web-exposed services.** Container health is
   not proof the web UI works. From nerv, hit the service through Traefik (use
   `--resolve` so you don't depend on DNS):
   - Reachable / routed: `curl -sk --resolve <SERVICE>_HOST:443:127.0.0.1 -o /dev/null -w "%{http_code}\n" https://<service>.geo-front.net/`
   - If auth is enabled: confirm an unauthenticated request is redirected or
     rejected (3xx/401), valid credentials are accepted, and **bad credentials
     are refused (401)**. Report the result.

9. **Persist the profile only if the service should be always-on.** Gating
   behind a profile keeps it off by default. If the human wants it to start with
   the whole stack, append the profile to `COMPOSE_PROFILES` in the server `.env`
   (don't clobber the existing list):
   `ssh gendo@nerv "cd ~/magi && grep -q '^COMPOSE_PROFILES=.*\b<service>\b' .env || sed -i '/^COMPOSE_PROFILES=/s/\$/,<service>/' .env"`
   Otherwise leave it on-demand (`COMPOSE_PROFILES=<service> docker compose up -d <service>`).

## Abort / Rollback

If the service cannot be made healthy (5 iterations exhausted) or the human calls
it off, back the change out cleanly rather than leaving a half-deployed service:

1. On nerv: `docker compose stop <service> && docker compose rm -f <service>`,
   then `git checkout master && git pull origin master` and bring the rest of the
   stack back to the known-good state (`docker compose up -d`).
2. Remove any orphaned server-side config dir only if **you** created it this
   session and it holds nothing the human wants to keep (confirm first).
3. Locally: `git checkout master`, delete the feature branch
   (`git branch -D feat/add-<service>`), and close/delete the PR and remote branch
   if one was opened.
4. Revert the server `.env` edits you made (e.g. remove the `<service>` entry from
   `COMPOSE_PROFILES` and any `<SERVICE>_HOSTNAME` line you added).

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
- **Never complete a run that generated a human-facing credential without
  verifying it was written to the Vaultwarden service vault**, or explicitly
  handing it to the human and receiving confirmation it was stored.
- **Never store anything in the Vaultwarden service vault other than
  machine-generated service credentials.** It is scoped deliberately; a personal
  secret placed there breaks the isolation it exists to provide.
- **Never read from, write to, or attempt to unlock the human's personal
  Vaultwarden vault.**
- **Never move, copy, commit, or echo the contents of
  `~/.config/bw-service-pass`.**
- **Never disable or weaken another service's healthcheck.**
- **Never force-merge past unresolved high-level review warnings.**
- **Never exceed 5 troubleshooting iterations or 2 review rounds without
  escalating.**
- **If `docker compose config` ever fails, fix that before anything else** — a
  broken config breaks variable resolution for the whole stack.
