# Contributing to airlock

Thanks for your interest in improving **airlock**. This is a small, security-critical
developer tool: a disposable, hardened sandbox for running untrusted code and
deploying with throwaway logins. Because its whole job is containment, the bar for
changes is a little different from a typical utility — a subtle regression here doesn't
just cause a bug, it can quietly hand someone's SSH keys or cloud credentials to
malicious code.

So the one rule that dominates everything else:

> **Never weaken the isolation model without saying so, explaining why, and proving the
> guarantee still holds.**

Everything below is in service of that.

---

## Ground rules

- **Security first.** If a change trades safety for convenience, it needs an explicit
  justification and a way to opt in — not a silent default. See
  [The security boundary](#the-security-boundary--do-not-regress).
- **macOS-first.** airlock targets a Mac host (it reads the Claude token from the macOS
  **Keychain** via `security`, and `setup.sh` installs Docker Desktop). Linux/Windows
  portability is welcome as a contribution, but keep the Mac path working.
- **It's personal tooling.** Maintained by one person in spare time. Small, focused
  contributions get reviewed faster than large rewrites. Open an issue to discuss
  anything big before you build it.
- **Be honest in docs.** The README is candid about what airlock does *not* protect
  against. Keep that spirit — don't oversell a guarantee.
- This project follows the [Code of Conduct](./CODE_OF_CONDUCT.md). By participating you
  agree to uphold it.

---

## Ways to contribute

- **Bug reports** — something behaves differently from the README, a mode fails to
  launch, the banner/status lies about state, etc.
- **Security issues** — see [Reporting a vulnerability](#reporting-a-vulnerability) below;
  please don't open a public issue for anything that breaks isolation.
- **Docs** — clarifications, fixed examples, better explanations of the threat model.
- **New deploy CLIs / tooling** in the sandbox image (kept lean and offline-friendly).
- **New whitelist entries** — held to a specific bar; see
  [Changing the egress whitelist](#changing-the-egress-whitelist).
- **Portability** — Linux/Windows host support, alternative Docker engines (Colima,
  Rancher Desktop), etc.
- **Tests / verification tooling** — there's currently only the smoke test in
  `setup.sh`; more automated isolation checks would be very welcome.

---

## Reporting a bug

Open a GitHub issue with:

- your macOS + chip (`uname -m`), Docker Desktop version, and airlock mode (`run` /
  `dev` / `sbx`);
- the exact command, the banner it printed, and what you expected vs. saw;
- relevant output from `airlock status`, `airlock blocked`, or `airlock logs`.

## Reporting a vulnerability

If you find a way to **escape the sandbox, read host files that shouldn't be reachable,
or exfiltrate data past the whitelist**, please do **not** file a public issue. Report it
privately to the maintainer (see the profile at
[github.com/syedtasavour](https://github.com/syedtasavour)) with a description and, ideally,
a minimal reproduction. Give a reasonable window to fix it before any public disclosure.

Note the residual risks the README already documents (shared kernel for `run`/`dev`, the
Claude token being present in `run` mode, whitelisted domains being trusted) are *known*
trade-offs, not bugs — but a concrete way to abuse them, or a way to tighten them without
breaking usability, is a great contribution.

---

## Development setup

Prerequisites:

- **Docker Desktop** running (the `sbx` microVM mode needs **4.58+** for
  `docker sandbox`).
- **macOS** — for the Keychain-based Claude login sync. On another OS the sandbox still
  builds, but credential seeding won't work as written.

Get a working copy:

```bash
git clone https://github.com/syedtasavour/Airlock.git
cd Airlock
bash setup.sh
```

`setup.sh` is idempotent. It checks Docker, creates `.secrets/` + `.logs/`, adds the
`airlock` shell alias, builds both images, and **self-tests that the egress whitelist
blocks a random host** (`example.com` must be denied). Re-run it any time.

What's git-ignored (never commit these): `.secrets/` and any `*.credentials.json` (the
exported Claude token), `.logs/`, and `*.log`. Double-check `git status` before you push.

---

## The security boundary — do not regress

These are the invariants that make airlock a sandbox rather than "just a container."
A PR that changes any of them **must call it out in the description and explain why the
guarantee still holds.** Reviews will block on unexplained changes to this list.

| Invariant | Where it lives | Why it matters |
|---|---|---|
| Read-only root filesystem | `read_only: true` in `docker-compose.yml` | Untrusted code can't persist or tamper with the image |
| All Linux capabilities dropped | `cap_drop: ALL` | Removes most privileged-operation escape routes |
| No privilege escalation | `security_opt: no-new-privileges:true` | `setuid` can't regain root |
| Non-root user | `USER node` in `Dockerfile` | Nothing runs as root inside |
| Ephemeral, RAM-backed home | `tmpfs: /home/node`, `/tmp` | Session state (incl. token refreshes) is wiped on exit |
| Only the project is mounted | `${WORKSPACE}:/workspace` volume | Untrusted code can't reach `~/.ssh`, `.env`, cloud creds |
| Untrusted mode has no direct internet | `networks: [isolated]` (`internal: true`) | No gateway → egress is only possible via the proxy |
| Egress forced through the proxy | `HTTP(S)_PROXY=http://proxy:8888` | Nothing bypasses the filter |
| Default-deny whitelist, incl. HTTPS | `FilterDefaultDeny Yes`, `FilterURLs Off` in `tinyproxy.conf` | Only listed hosts resolve; the `CONNECT` host is filtered too |
| TLS ports only | `ConnectPort 443` / `563` | No tunnelling to arbitrary ports |
| No history/secrets in `run` mode | only `.credentials.json` mounted (not `~/.claude`) | Past conversations stay off the untrusted box |
| Resource caps | `mem_limit`, `cpus`, `pids_limit` | Blunts fork bombs / runaway builds |

If you're **loosening** something on purpose (e.g. a tool genuinely needs a capability),
make it opt-in via an environment variable in `docker-compose.yml` — mirror the existing
`AIRLOCK_SEED_CLAUDE` pattern — and default it to the safe value.

---

## Changing the egress whitelist

The whitelist in `proxy/filter` is the single most security-sensitive file. Every domain
you add is a host that untrusted code can now talk to — and several already-listed hosts
(GitHub, `*.googleapis.com`, `*.workers.dev`, `*.herokuapp.com`) can serve
attacker-controlled content, so additions widen the potential exfiltration surface. Keep
the list tight.

Bar for adding a domain:

- It's needed by a **common, mainstream** ecosystem — a package registry, a widely used
  deploy target — not a single niche service.
- Prefer the **narrowest** rule that works. One extended-regex per line, matched against
  the destination **host**.
- **Anchor every rule.** Rules are a regex *search*, so anchor the end with `$` at
  minimum, and mirror the existing style in the file:
  - to allow a bare apex domain (`example.com`): `^example\.com$`
  - to allow its subdomains (`api.example.com`): `\.example\.com$`
  - most registries/deploy targets need **both** lines.
  - `airlock allow <domain>` writes only the **subdomain** form, so if the bare apex host
    must resolve, add the `^example\.com$` line by hand.
- After editing, **rebuild** so the change is baked into the proxy image, and verify:

```bash
airlock rebuild
airlock run
#   inside: curl -sSI https://<new-host>        -> should connect
#   inside: curl -sSI https://tracker.evil.test -> should be blocked
airlock blocked            # confirm denials are logged as expected
```

Keep the human-readable view honest: the CLI reverse-maps these regexes to `*.domain`
form for the banner and `airlock status`. If you change the filter's format, update
`readable_whitelist()` in the `airlock` script so the displayed list still matches reality.

---

## Building & testing your change

There is **no automated test suite yet** — so manual verification matters, and your PR
should say what you checked.

Rebuild after any change to the `Dockerfile`, compose file, or proxy:

```bash
airlock build            # normal build
airlock rebuild          # from scratch, no cache (use when in doubt)
```

Then exercise the modes your change touches. At minimum, confirm the boundary still holds
from inside `airlock run`:

```bash
# filesystem: only the project is visible, root is read-only, you're not root
ls /                     # no host home, no ~/.ssh
touch /nope              # must FAIL (read-only root)
id                       # uid should be 1000 (node), not 0

# capabilities dropped (should be empty / minimal)
grep CapEff /proc/self/status

# egress: whitelisted host works, unlisted host is blocked
curl -sSI https://registry.npmjs.org   # connects
curl -sSI https://example.com          # blocked
```

The canonical smoke test is the one in `setup.sh` — a fresh `example.com` request from an
untrusted box **must** be denied. If your change could affect networking or isolation,
re-run `bash setup.sh` and make sure the self-test still passes.

For `sbx` changes you'll need Docker Desktop 4.58+; verify the whitelist is applied to the
microVM (non-listed hosts return `403`) and that `airlock sbx down` / `--fresh` behave.

---

## Coding conventions

Match the existing style — the codebase is deliberately, heavily commented with the
*reasoning* behind each decision (why a binary lives in `/opt`, why pnpm is baked in, why a
network is `internal`). Keep that up; the comments are half the documentation.

**Bash (`airlock`, `setup.sh`, `entrypoint.sh`)**
- Start scripts with `set -euo pipefail`.
- Quote variable expansions (`"$VAR"`), especially paths.
- Reuse the existing helpers (`hr`, the color vars, `readable_whitelist`, `dc`) rather
  than reinventing them; keep colors gated on `[ -t 1 ]` so piped output stays clean.
- Keep the CLI dependency-light — it should run on a stock macOS shell.

**Dockerfile**
- Pin versions (the base image, nvm, etc.) — reproducibility is a feature.
- Anything that must survive the **read-only root** + **tmpfs home** goes in a persistent,
  root-owned location like `/usr/local` or `/opt`, installed at build time. Prefer real
  baked binaries over "fetch latest at runtime" (that breaks behind the whitelist).
- Clean package caches (`rm -rf /var/lib/apt/lists/*`, `npm cache clean --force`).

**Docs & UX**
- When you add or change a command, update **all three** surfaces: the README, the
  `print_help` text, and the launch banner / `airlock status` if relevant. They should
  never disagree.
- Follow the README's tone: direct, concrete, honest about limits.

---

## Commits & pull requests

- Keep PRs small and focused — **one logical change per PR.**
- Write clear commit messages (imperative mood: "add cargo registry to whitelist").
- In the PR description, include:
  - **what** changed and **why**;
  - **how you tested it** — especially any isolation checks you ran;
  - an explicit note if it touches anything in
    [The security boundary](#the-security-boundary--do-not-regress).
- Rebase/clean up obviously-WIP commits before requesting review.

PR checklist:

- [ ] Rebuilt (`airlock rebuild`) and exercised the affected mode(s).
- [ ] If networking/isolation was touched: `setup.sh` self-test still passes.
- [ ] No secrets, tokens, or `.log` files committed (`git status` is clean of ignored paths).
- [ ] README, `--help`, and banner/status updated if behavior or commands changed.
- [ ] Security-boundary changes are called out and justified in the description.

---

Thanks again for helping make airlock better — and safer.
