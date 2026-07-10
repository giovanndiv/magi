# Playbook: Configure a Config-Driven Service in the MAGI Stack

> This file is a spec written to **you, Claude Code**, as the executor. Read
> `CLAUDE.md` first, then follow these steps in order to configure a single
> service whose value lives in a hand-authored config file. Stop and escalate to
> the human at every point marked **STOP**. This playbook is the companion to
> `ADD_SERVICE.md`: it may be run standalone, or as a second pass after
> `ADD_SERVICE.md` has already deployed the compose block. A first run here can
> **do things** (inject torrents, call external APIs, move or link data) — treat
> the first execution as human-gated, never autonomous.

## Purpose

This playbook covers config-driven services: the container is trivial to stand
up, but the value lives entirely in a hand-authored config file that (a)
contains secrets or environment-specific values you must never invent, and (b)
drives actions with real blast radius — injecting torrents, calling external
APIs, moving or linking data. The core split: **you own the scaffold and
deploy; the human owns the config content and the go/no-go on anything
destructive.** You get an inert, correctly-mounted, correctly-permissioned
container in place, then **STOP** and hand the human a filled-in config template
with every secret and risky value flagged; you wait for them to supply and
approve; then you do a gated, human-watched first run. It works standalone or as
a second pass after `ADD_SERVICE.md` — so it always **scaffolds-or-verifies**
rather than assuming a fresh add.

## When to use this vs ADD_SERVICE

- **Use `ADD_SERVICE.md`** when the service's configuration is derivable from
  stack conventions — PUID/PGID, TZ, a `/config` volume, Traefik labels, a
  homepage block. Everything the service needs comes from the patterns already
  in the repo.
- **Use this playbook** when standing the service up means writing a
  hand-authored config file containing **secrets**, **environment-specific
  values**, or **safety-critical behavior settings**.

**The tell:** if bringing the service to life requires authoring a config file
with API keys, external URLs, data paths, or destructive-action settings, it is
a CONFIGURE_SERVICE job — even if the compose block itself is trivial.

**Worked example — cross-seed** (referenced throughout this file): the compose
block is four lines, but the real work is `config.js`: the **Prowlarr Torznab
API key** (secret), the **qBittorrent connection over `http://vpn:8080`** (the
VPN quirk — cross-seed reaches qBittorrent through the `vpn` container, not
`qbittorrent:8080`), the **`dataDirs` paths** (environment-specific, must be
mounted), **`matchMode`** (safety: strict vs. flexible matching), and
**`skipRecheck`** (safety: whether injected torrents are rechecked). None of
that is derivable from conventions, and getting it wrong injects bad data or
hammers a tracker.

## Input Contract

The human provides:

1. **The service name** — used for the container name, branch, and config path.
2. **The upstream source** — the Docker image reference or the project's GitHub
   repo, so you can research the config format.
3. **Acknowledgment that this service needs a human-authored config** — i.e.
   that this is the right playbook.

**If the compose block already exists** because `ADD_SERVICE.md` ran first, the
input may be just the service name — you will verify the scaffold rather than
build it.

If anything is missing or ambiguous, **STOP and ask** before doing anything.

## Config Classification (the core deliverable)

Before writing anything, research the upstream config format from the official
docs, then classify **every field the config will need** into three categories.
This classification is the deliverable that makes the handoff safe — present it
clearly to the human.

1. **SECRETS — never invent or guess.** API keys, passkeys, passwords, tokens.
   Leave these as `REPLACE_ME` placeholders for the human to fill server-side.
   _(cross-seed: the Prowlarr Torznab API key.)_

2. **ENVIRONMENT-SPECIFIC but non-secret — you MAY propose from stack
   conventions, for human approval.** Internal service URLs, the
   `http://vpn:8080` qBittorrent quirk, data paths, container hostnames. Propose
   concrete values, but mark them as **proposed, not final** — the human
   confirms. _(cross-seed: `torznab` URL pointing at `http://prowlarr:9696`, the
   `qbittorrent` client URL as `http://vpn:8080`, `dataDirs` under `/data`.)_

3. **SAFETY / BEHAVIOR — the human must consciously choose.** Match strictness,
   recheck on/off, delete-vs-pause actions — anything that determines blast
   radius. **Default each to the SAFEST value** with an inline comment
   explaining why, and flag every one for explicit human confirmation.
   _(cross-seed: `matchMode` defaulted to the stricter option;
   `skipRecheck: false` so injected torrents are verified before seeding, with a
   comment saying why the safe default was chosen.)_

## Preconditions (verify before starting)

Inherit `ADD_SERVICE.md`'s preconditions — run these first, and **STOP** with
the stated message if any fails:

- **Passwordless SSH to nerv works.** Test: `ssh gendo@nerv "echo connected"`.
- **Working tree is clean, on `master`, up to date with origin.** Test:
  `git status`, `git branch --show-current`, `git fetch && git status`.
- **The server config is currently valid.** Test:
  `ssh gendo@nerv "cd ~/magi && docker compose config"`. If it fails, **STOP** —
  a broken config breaks variable resolution globally.

**Add one precondition specific to config-driven services:** if the config will
drive **linking or data operations**, verify — **before writing the config, not
at runtime** — that the container's mounts expose **every path the config will
reference**, and that any paths that must be hardlinked sit on a **single shared
mount**. This is the same cross-device rule as the qBittorrent/cross-seed
hardlink fix: cross-seed's `dataDirs` and the torrent client's save path must
live under the same `/data` mount, or hardlinking silently fails across devices.
**If a needed path is not mounted, fix the mount first** — and note that a mount
change is an `ADD_SERVICE.md`-style **important-change PR** (it affects running
services), not a docs change.

**Mount breadth, not just mount presence.** The mount must be wide enough for
the tool to *see* everything it matches against — not only where it writes. In
the real cross-seed run the compose block mounted only `${DOWNLOAD_ROOT}`
(`/data/torrents`) instead of the full `${DATA_ROOT}` (`/data`), so cross-seed
was blind to the media library it needed to cross-seed against — the config's
`dataDirs` referenced paths the mount never exposed, and this surfaced only at
runtime. Enumerate every path the config will point at and confirm each is
visible *inside the container* (`ssh gendo@nerv "docker compose run --rm
<service> ls <path>"` or equivalent) **before writing the config**.

## Scaffold Steps (your lane)

1. **Branch.** Check out a new branch named `feat/configure-<service>`.

2. **Add or verify the compose block.** If `ADD_SERVICE.md` already deployed it,
   verify it; otherwise add it following `ADD_SERVICE.md`'s conventions
   (container name, `restart: always`, profile gating for non-core services, the
   `nerv` default network). Do not assume a fresh add — **scaffold-or-verify.**

3. **Verify the mounts are correct**, applying the **same-mount rule** above for
   any linking service — config dir under `${CONFIG_ROOT:-.}/<service>:/config`,
   and every data path the config references exposed under a shared `/data`
   mount.

4. **Create the config and data/link dirs with correct `1000:1000`
   ownership.** Note the chain from `ADD_SERVICE.md`: when the container first
   starts, Docker creates the host-side bind-mount dir owned by **root**, and
   `gendo` has **no passwordless sudo** and the in-session `!` prefix has **no
   TTY**. So when a chown is needed, ask the human to run it in a real terminal:
   `ssh -t gendo@nerv "sudo chown -R 1000:1000 ~/magi/<service>"`.

5. **Ensure the container CAN start but is effectively inert without config.**
   The scaffold should stand up cleanly; it should do nothing meaningful until a
   real config is present.

6. **Do NOT write the real config.** Write a **TEMPLATE** at the config path
   with:
   - placeholders and inline comments explaining each field,
   - **secrets as `REPLACE_ME`**,
   - every **safety setting defaulted to its conservative value** with a comment
     saying why.

   This template, not a finished config, is what you hand off.

## Handoff Checkpoint (the heart of this playbook)

**STOP.** This is the pivot from your lane to the human's. Present, together:

- **The template** you wrote at the config path.
- **The config classification** — which fields are secret (`REPLACE_ME`), which
  the human must confirm, and which safety values the human must choose.
- **Your proposed values** for the environment-specific non-secret fields
  (marked as proposed, not final).

The human then, **on the server**:

- **fills the secrets** — secrets are entered server-side only, **never in git,
  never echoed into a tracked file**, and
- **approves the risky choices** (the safety/behavior settings and the proposed
  environment values).

**Credential persistence checkpoint.** If, **at this point — before any run**,
you or the human have generated or set a credential that will be needed later (a
password the human supplies, or one you generate and write into the config —
e.g. a hashed auth password), that credential **must be persisted to the
Vaultwarden service vault before proceeding**. Do not strand a secret.
(Credentials the *service itself* mints on its **first run** don't exist yet —
those are caught by this same procedure at the Gated First Run step below,
before promotion to daemon mode.)

Run the following over SSH on nerv. This writes to a **dedicated Vaultwarden
service account** whose vault holds **only** machine-generated service
credentials — it is not the human's personal vault:

- **Confirm prerequisites.** `command -v bw` must succeed, and the Vaultwarden
  service must be reachable (`docker compose ps vaultwarden` shows healthy — the
  healthcheck comes from the upstream Vaultwarden image, not this repo's compose
  file). If either fails, **STOP** — hand the human the credential and the
  suggested entry name, and wait for explicit confirmation it has been stored
  before proceeding.
- **Unlock fresh** — `BW_SESSION` does not persist across shells, so every
  invocation unlocks, acts, and locks:

      export BW_SESSION=$(bw unlock --passwordfile ~/.config/bw-service-pass --raw)

- **Create the entry** using the naming convention `magi/<service>`, with the
  username, the generated password, and a note containing the service URL and
  creation date. Build it with `bw get template item`, populate the fields with
  `jq`, then `bw encode | bw create item`.
- **Verify the write by reading it back:** `bw list items --search magi/<service>`
  must return the entry. A silent failed write is the exact failure this
  checkpoint exists to prevent — do not assume success.
- **Lock when done:** `bw lock`.
- **Never** echo the plaintext credential into a tracked file, a log, a commit
  message, or a PR comment.

**Wait for explicit human confirmation** that secrets are filled, safety values
approved, and any generated credential persisted (and verified) to the service
vault, before continuing.

## One-time vs Steady-state

Before the first run, **ask the human whether this is a one-time / recovery
configuration or the permanent config.** Some configure tasks have a "do once,
then reconfigure for ongoing" shape.

**cross-seed is the example:** a one-shot data-based search (populating
`dataDirs` and running a single search pass to cross-seed an existing library)
is a **recovery** config you then **strip out** before switching to `daemon`
mode for ongoing operation. If the task is one-time, **plan the
strip-and-reconfigure follow-up and note it** — do not leave the recovery config
in place as the permanent steady-state config.

## Gated First Run (replaces ADD_SERVICE's autonomous test loop)

Because a first run here can **do things** (inject, call APIs, touch data), it
must be **human-watched**. There is **no autonomous troubleshooting loop** that
re-runs a destructive action.

1. **Prefer the service's dry-run or one-shot mode for the first execution** —
   never straight to daemon. Use a one-shot invocation, e.g.
   `docker compose run --rm <service> <one-shot-command>` (this is exactly how
   the cross-seed recovery **search** is run — a single search pass, not
   `command: daemon`).

2. **The human watches the output** and confirms the run did what was intended
   (e.g. cross-seed matched the expected torrents and injected only those, with
   no unexpected tracker calls or data moves).

3. **Verify the intended END STATE, not just that execution completed.** A run
   that reports success can still leave the wrong end state. In the real
   cross-seed run, injected torrents rechecked to 100% and showed as "done" — so
   execution looked complete — but they had silently stopped and were **not
   announcing to the tracker**; they only actually seeded after a manual
   force-resume. Decide the concrete end-state check up front (torrents active
   and announcing / actually seeding, the API call produced the intended effect,
   the data landed where intended) and have the human confirm *that*, not merely
   that the tool exited without error.

4. **Only promote to persistent / daemon mode after that confirmation** — flip
   to `command: daemon` (or `restart: always` / profile-persist per
   `ADD_SERVICE.md`) once, and only once, the human approves the observed first
   run.

5. **Persist any credential the first run itself produced — a precondition on
   that promotion.** If the observed first run **minted, printed, or otherwise
   produced** any credential the human will need later (an admin password shown
   at first boot, an API token the service generated, a session secret in the
   logs), re-invoke the credential persistence procedure **now** — follow the
   Credential persistence checkpoint procedure in the Handoff Checkpoint section
   above. Do **NOT** promote to daemon mode until that credential is persisted
   and the write is verified. This is the credential that did not yet exist at
   the Handoff Checkpoint, so it is caught here instead.

6. **If the first run fails: do NOT loop.** Read the logs, form **ONE**
   hypothesis, present it and the proposed fix to the human, and **wait**. Never
   re-run a destructive action autonomously to "see if it works now."

## PR, Review, Merge

Same as `ADD_SERVICE.md` — the **compose/scaffold changes** go through the
**important-change workflow** with the `@claude` review poll and the **two-round
cap**. What is tracked vs. not:

- **Tracked (goes in the PR):** the compose block, the profile/env-example
  entries, and the **`.gitignore` entry for the service's config dir** (matching
  the existing `/cross-seed`, `/sonarr`, `/cleanuparr` entries) so the
  secret-bearing config can never be committed.
- **Never tracked:** the **real config file** — it is server-side and
  gitignored. Confirm the config dir is gitignored **as part of this playbook**.

Steps: commit and push `feat/configure-<service>`, then:

1. `gh pr create --base master --head feat/configure-<service> --title "feat: configure <service>"`
2. `gh pr comment --body "@claude review"`
3. Poll until the bot finishes:

       while true; do
         COMMENT=$(gh pr view --json comments --jq '.comments[-1].body')
         if echo "$COMMENT" | grep -q "Claude finished"; then
           echo "Review complete:"; echo "$COMMENT"; break
         fi
         sleep 30
       done

4. Read the review. Resolve blocking warnings within scope, push, and re-run the
   review + poll **once** more. **After two rounds:** clean → **Merge**;
   warnings remain → **STOP** and escalate. Do not loop a third time.
5. **Merge:** `gh pr merge --merge`, then
   `git checkout master && git pull origin master`, delete the local and remote
   branch, and on nerv `ssh gendo@nerv "cd ~/magi && git pull origin master"`.
   The real config on the server is untouched by the merge (it's gitignored).

## Guardrails (must never do without stopping to ask)

- **Never modify any service other than the one being configured** — no edits to
  other services' configs or compose blocks.
- **Never invent, guess, or fill a secret yourself.**
- **Never finalize a safety-critical value without explicit human
  confirmation** — default to conservative and flag it.
- **Never promote to continuous / daemon mode without a successful
  human-observed first run.**
- **Never run a config-driven action with data or external blast radius without
  explicit human go.**
- **Never write real secrets, keys, passwords, or private tracker details to any
  tracked file.**
- **Never leave a generated human-facing credential unstored** — persist it to
  the Vaultwarden service vault and verify the write, or explicitly hand it to
  the human and receive confirmation it was stored, before completing the run.
- **Never store anything in the Vaultwarden service vault other than
  machine-generated service credentials.** It is scoped deliberately; a personal
  secret placed there breaks the isolation it exists to provide.
- **Never read from, write to, or attempt to unlock the human's personal
  Vaultwarden vault.**
- **Never move, copy, commit, or echo the contents of
  `~/.config/bw-service-pass`.**
- **Never touch the storage layout, the mergerfs/snapraid data mounts, or the
  qBittorrent / VPN (`network_mode: "service:vpn"`) coupling.**
- **No autonomous re-run loop on a destructive first run** — one hypothesis,
  then escalate.
