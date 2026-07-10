# MAGI Playbooks

This directory is a set of specs that **Claude Code** follows to operate on the
MAGI stack, each with human-escalation gates built in — you execute the steps;
the human owns the go/no-go at every point marked **STOP**.

## Routing — which playbook

Match the intent to the playbook:

- **Adding a new service** whose configuration is derivable from stack
  conventions (PUID/PGID, TZ, a `/config` volume, Traefik labels, a homepage
  block) → **[`ADD_SERVICE.md`](ADD_SERVICE.md)**.
- **Removing / decommissioning** an existing service →
  **[`REMOVE_SERVICE.md`](REMOVE_SERVICE.md)**.
- **Configuring a service** whose value lives in a hand-authored config file
  containing secrets, external API calls, or destructive actions →
  **[`CONFIGURE_SERVICE.md`](CONFIGURE_SERVICE.md)**. This can run **standalone**,
  or as a **second pass after `ADD_SERVICE.md`** has deployed the compose block.

The tell for CONFIGURE vs ADD: if bringing the service to life requires authoring
a config file with API keys, external URLs, data paths, or destructive-action
settings, it is a CONFIGURE job even when the compose block itself is trivial.

## Shared conventions

All three playbooks inherit the following. They are stated once here so the
individual playbooks can rely on them rather than re-explaining each one:

- **Passwordless SSH to nerv.** `ssh gendo@nerv "echo connected"` must succeed
  before any playbook starts; if it prompts for a password, STOP.
- **Clean tree, on `master`, up to date with origin.** Verified with
  `git status` / `git branch --show-current` / `git fetch && git status` before
  starting.
- **Valid server config.** `ssh gendo@nerv "cd ~/magi && docker compose config"`
  must succeed. **If it fails, fix that first** — a broken config breaks variable
  resolution for the whole stack, so no playbook proceeds until it is valid.
- **`@claude` review loop with a two-round cap.** After opening the PR, trigger
  `@claude review` and poll until the bot posts "Claude finished". Resolve
  blocking warnings within scope, push, and re-run the review **once** more.
  After two rounds: clean → merge; warnings remaining → **STOP** and escalate.
  Never loop a third time.
- **Root-owned bind mount → `ssh -t ... sudo chown`.** When a container first
  starts, Docker creates the host-side bind-mount dir owned by **root**; `gendo`
  has **no passwordless sudo** and the in-session `!` prefix has **no TTY**. So
  any chown must be run by the human in a real terminal:
  `ssh -t gendo@nerv "sudo chown -R 1000:1000 ~/magi/<service>"`.
- **Secrets never enter tracked files.** Real secrets live only server-side and
  are entered by the human on nerv — never invented, never echoed into a tracked
  file, never committed. (See **Secrets & exposure** below for the full rule.)

## Secrets & exposure

This repository is **public on GitHub**, so secret hygiene is a hard constraint,
not a preference. Every playbook inherits this rule:

- **No real secrets in tracked files — ever.** No API keys, passwords, tokens,
  private/LAN IPs, or private-tracker passkeys/announce URLs may be committed to
  any tracked file.
- **Every secret-bearing config dir is gitignored.** Add the service's config dir
  to `.gitignore` (matching the existing `/sonarr/`, `/cross-seed/`,
  `/cleanuparr/` entries) so its contents can never be committed.
- **Every PR must pass the repository's secret scan before merge.** This is
  enforced by an **active CI secret-scanning check** — **gitleaks**, run on every
  pull request (and on push to `master`) by
  [`.github/workflows/gitleaks.yml`](../../.github/workflows/gitleaks.yml) against
  the rules in [`.gitleaks.toml`](../../.gitleaks.toml). Any detected secret fails
  the check, so a leaking PR cannot be merged. This is not left to reviewer
  diligence — a human reviewer missing a leaked key is not the safety net; the
  automated scan is. The ruleset extends the gitleaks defaults with rules for this
  repo's exposure surface (tracker passkeys/announce URLs, *arr/Prowlarr API keys,
  and Tailscale/RFC1918 IPs).

**Persisting generated service credentials.** When a playbook run generates a
credential a human will need later (an admin password, an API token, an auth
secret), it is persisted to a **dedicated Vaultwarden service-account vault** via
the `bw` CLI on nerv, under the naming convention `magi/<service>`. That vault is
scoped to **machine-generated service credentials only** and is isolated from the
human's personal Vaultwarden vault — nothing personal goes in it, and playbooks
never touch the personal vault. Unattended unlock uses a password file at
`~/.config/bw-service-pass` (chmod 600, owned by `gendo`, living outside the repo
and never tracked). `BW_SESSION` does not persist across shells, so every write
unlocks fresh, creates the entry, verifies it by reading it back, and locks. The
full procedure lives in `ADD_SERVICE.md` and `CONFIGURE_SERVICE.md`.

The procedure is **intentionally restated in full in each playbook** rather than
referenced — a spec an executor follows step-by-step should not require reading
another file mid-run, and indirection in a spec is worse than duplication. The
cost is that any change to the procedure (the `magi/<service>` naming
convention, the `bw unlock` invocation, the read-back verification step) **must
be applied to every copy** — `ADD_SERVICE.md`, `CONFIGURE_SERVICE.md`, and this
summary.

## Capability map

| Playbook | Purpose | Branch convention |
|---|---|---|
| [`ADD_SERVICE.md`](ADD_SERVICE.md) | Add a new service whose config is derivable from stack conventions. | `feat/add-<service>` |
| [`REMOVE_SERVICE.md`](REMOVE_SERVICE.md) | Cleanly decommission an existing service. | `chore/remove-<service>` |
| [`CONFIGURE_SERVICE.md`](CONFIGURE_SERVICE.md) | Configure a service whose value lives in a hand-authored, secret-bearing config file. | `feat/configure-<service>` |

## Future playbooks (roadmap — not yet built)

Candidates already discussed, kept here so the set has a direction:

- **VPN_RECOVERY** — recover the qBittorrent/Gluetun VPN subsystem. Highest-value
  next playbook given the Gluetun control-server auth migration in the backlog.
- **TROUBLESHOOT_SERVICE** — diagnose an unhealthy service from its logs.

_(CI secret-scanning with gitleaks — previously listed here — is now **active**;
see the **Secrets & exposure** section above.)_
