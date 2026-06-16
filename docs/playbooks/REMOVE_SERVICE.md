# Playbook: Remove a Service from the MAGI Stack

> This file is a spec written to **you, Claude Code**, as the executor. Read
> `CLAUDE.md` first, then follow these steps in order to cleanly decommission a
> single existing service from the MAGI Docker Compose stack. Stop and escalate
> to the human at every point marked **STOP**. This is the inverse of
> `ADD_SERVICE.md`. Removal can destroy data — treat every delete as
> irreversible and confirm before acting.

## Purpose

This playbook lets you remove one existing service end to end: confirm scope,
find everything the service touches (compose block, env vars, `.env.example`,
`.gitignore`, homepage labels, `COMPOSE_PROFILES`, server-side config dir,
subdomain), stop and remove the container on `nerv`, strip the tracked
references, open a PR with the automated `@claude` review, and merge — while
stopping to escalate at the defined checkpoints. Operate within the target
service's scope only; never touch unrelated services, storage, or the VPN, and
never delete data the human has not explicitly authorized.

## Input Contract

The human provides only:

1. **The service name** to remove (the `container_name` / compose service key).

Optionally, the human states **whether to delete the service's persisted config
dir** (its data). If they do not, you **must ask** before deleting anything on
disk — default to **keeping** the data.

If the named service does not exist in `docker-compose.yml` (or a subdirectory
compose file), or the name is ambiguous, **STOP and ask**.

## Preconditions (verify before starting)

Run these checks first. If any fails, **STOP** with the stated message.

- **Passwordless SSH to nerv works.** Test: `ssh gendo@nerv "echo connected"`.
  If it prompts for a password or fails, **STOP** and tell the human to set up
  the SSH key.
- **Working tree is clean, on `master`, up to date with origin.** If not,
  **STOP** and ask how to proceed.
- **The server config is currently valid.** Test:
  `ssh gendo@nerv "cd ~/magi && docker compose config"`. If it fails, **STOP** —
  fix the broken config before removing anything.

## Dependency & Safety Check (do this BEFORE touching anything)

Removal is dangerous when another service depends on the target. **STOP and
escalate** if any of these are true — do not proceed without explicit human
approval:

1. **The target is shared infrastructure.** Never remove `traefik`, `vpn`
   (Gluetun), `homepage`, `adguardhome`, `watchtower`, or `autoheal` as part of
   a routine service removal — these underpin the whole stack.
2. **Another service depends on the target.** Grep the repo and escalate if hits
   exist outside the target's own block:
   - `grep -rn "<service>" docker-compose.yml */docker-compose.yml` — look for
     `depends_on: <service>`, `network_mode: "service:<service>"`, widget URLs
     like `http://<service>:<port>`, or `*_URL` env references.
   - Specifically: **never** remove `vpn` while `qbittorrent` uses
     `network_mode: "service:vpn"`; **never** remove `prowlarr` without flagging
     that all *arr apps point at it.
3. **The target holds data the human may want.** App config/databases live under
   `${CONFIG_ROOT:-.}/<service>`. Media is **never** stored there (it lives under
   `${DATA_ROOT}`), but request history, settings, and credentials are. Confirm
   the keep-or-delete decision from the Input Contract before any disk deletion.

## Build Steps (tracked-file changes)

1. **Branch.** Check out a new branch named `chore/remove-<service>`.

2. **Inventory every reference** so nothing is left dangling. Search and note
   each hit:
   - `grep -rn "<service>" docker-compose.yml docker-compose.override.yml */docker-compose.yml`
   - `grep -rni "<service>" .env.example .gitignore`
   - The service's own block, its Traefik labels, its homepage labels, its
     `profiles:` entry.

3. **Remove the service block** from whichever compose file defines it. Remove
   only that block — leave surrounding services untouched.

4. **Remove its tracked env entries** from `.env.example`: the
   `<SERVICE>_HOSTNAME` line and any `<SERVICE>_API_KEY` / service-specific
   placeholders. Do **not** remove shared vars used by other services.

5. **Remove its `.gitignore` entry** (e.g. `/<service>/`) if one exists and no
   other service shares the path.

6. **Remove it from any documentation that enumerates services**: the
   `### Optional Services via Profiles` list and the active-services table in
   `CLAUDE.md`, and the backlog/runbook if it is listed there.

7. **Validate before pushing.**
   - **YAML sanity (always):** confirm the service is gone and the file still
     parses —
     `python3 -c "import yaml; d=yaml.safe_load(open('docker-compose.yml')); assert '<service>' not in d['services']; print('removed OK')"`.
   - **`docker compose config` (if Docker is available locally):** otherwise rely
     on the authoritative run on nerv below.

## Teardown on nerv (over SSH)

1. **Stop and remove the container** (do this before pulling the branch, so the
   running container is cleaned up regardless of compose state):
   `ssh gendo@nerv "cd ~/magi && docker compose stop <service> && docker compose rm -f <service>"`

2. **Back up the config dir before any deletion** (cheap insurance, even if the
   human said delete):
   `ssh gendo@nerv "cd ~/magi && [ -d <service> ] && tar czf /tmp/<service>-config-backup.tar.gz <service> && echo backed up to /tmp/<service>-config-backup.tar.gz"`

3. **Delete the config dir only if the human explicitly authorized it.** The dir
   may be **root-owned** (created by Docker), so deletion needs sudo in a real
   TTY — ask the human to run:
   `ssh -t gendo@nerv "sudo rm -rf ~/magi/<service>"`
   If the human wants to keep the data, **skip this** and leave the dir in place.

4. **Remove server `.env` entries** you or a prior session added for this
   service: drop the `<SERVICE>_HOSTNAME` / `<SERVICE>_API_KEY` lines, and remove
   the service from `COMPOSE_PROFILES` (don't clobber the rest of the list):
   `ssh gendo@nerv "cd ~/magi && sed -i '/^<SERVICE>_HOSTNAME=/d; /^<SERVICE>_API_KEY=/d' .env && sed -i '/^COMPOSE_PROFILES=/s/,\?<service>//' .env"`
   Then verify the `COMPOSE_PROFILES` line still looks right.

## PR and Review Loop

1. Commit and push the `chore/remove-<service>` branch.

2. Open the PR:
   `gh pr create --base master --head chore/remove-<service> --title "chore: remove <service>"`

3. Trigger the review: `gh pr comment --body "@claude review"`.

4. Poll until the GitHub Claude bot finishes:

       while true; do
         COMMENT=$(gh pr view --json comments --jq '.comments[-1].body')
         if echo "$COMMENT" | grep -q "Claude finished"; then
           echo "Review complete:"; echo "$COMMENT"; break
         fi
         sleep 30
       done

5. Read the review. If there are high-level / blocking warnings (e.g. a dangling
   reference the inventory missed), resolve them within the removal's scope,
   push, and re-run the review + poll **once** more.

6. **After two review rounds:** if clean, proceed to **Merge**; if warnings
   remain, **STOP** and escalate with a summary. Do not loop a third time.

## Merge

On a clean review:

1. `gh pr merge --merge`
2. `git checkout master && git pull origin master`
3. `git branch -d chore/remove-<service> && git push origin --delete chore/remove-<service>`
4. On nerv: `ssh gendo@nerv "cd ~/magi && git pull origin master"`
5. Confirm the rest of the stack is still healthy:
   `ssh gendo@nerv "cd ~/magi && docker compose config >/dev/null && echo CONFIG_OK && docker ps --format '{{.Names}} {{.Status}}' | grep -iE 'restarting|unhealthy' || echo 'none unhealthy'"`
6. Tell the human where the config backup lives (`/tmp/<service>-config-backup.tar.gz`)
   so they can delete it once they're confident the removal is complete.

## Guardrails (must never do without stopping to ask)

- **Never remove shared infrastructure** (`traefik`, `vpn`, `homepage`,
  `adguardhome`, `watchtower`, `autoheal`) as a routine removal.
- **Never remove a service another one depends on** without flagging the
  dependency and getting explicit approval — especially the `qbittorrent` ↔
  `vpn` (`network_mode: "service:vpn"`) coupling and `prowlarr` ↔ *arr apps.
- **Never delete the config dir, or anything under `${DATA_ROOT}`, without
  explicit human authorization.** Default to keeping data. Media is never in a
  service config dir — do not touch `${DATA_ROOT}`.
- **Always back up the config dir before deleting it.**
- **Never modify any service block other than the one being removed.**
- **Never remove shared `.env` / `.env.example` variables** used by other
  services.
- **Never exceed 2 review rounds without escalating.**
- **If `docker compose config` ever fails, fix that before anything else.**
