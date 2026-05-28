# AGENTS.md

> Behavioral rules for AI agents contributing to Dakota. Human contributors follow the same steps.

## Find something to work on

| Time available | Link |
|---|---|
| 30 min | [XS issues](https://github.com/projectbluefin/dakota/issues?q=is%3Aopen+label%3Aqueue%2Fagent-ready+label%3Asize%2Fxs+no%3Aassignee) |
| Half day | [S issues](https://github.com/projectbluefin/dakota/issues?q=is%3Aopen+label%3Aqueue%2Fagent-ready+label%3Asize%2Fs+no%3Aassignee) |
| Full day | [M issues](https://github.com/projectbluefin/dakota/issues?q=is%3Aopen+label%3Aqueue%2Fagent-ready+label%3Asize%2Fm+no%3Aassignee) |
| All | [Everything ready](https://github.com/projectbluefin/dakota/issues?q=is%3Aopen+label%3Aqueue%2Fagent-ready+no%3Aassignee+sort%3Acreated-asc) |

Comment `/claim` on an issue to take it. Actionadon assigns it and removes it from the pool. No PR activity in 7 days returns it automatically.

Dakota is a [BuildStream 2](https://buildstream.build/) project producing **Bluefin** — a bootc OCI desktop image built from source via freedesktop-sdk and gnome-build-meta. No RPMs. No dnf. BST elements only.

Dakota's primary design goal is a **built-in quality feedback loop** where users, contributors, agents, and hardware produce structured evidence that flows back into the next iteration. See [docs/feedback-loop.md](docs/feedback-loop.md).

---

## Mandatory gates

Non-compliance = automatic rejection.

**Read-First:** Read `README.md`, `AGENTS.md`, `.github/copilot-instructions.md`, and relevant `.github/skills/` files before modifying anything. Do not assume project structure or dependency patterns. Dakota keeps its instructions in-repo — do not rely on operator-local paths.

**Rate limit:** Max 4 open PRs at a time. If a PR is closed for quality, document the root cause on the closed PR before resubmitting.

**Operator accountability:** The human deploying the agent is responsible for all decisions. PR template checkbox: `[ ] I am using an agent and I take responsibility for this PR`

**Verification:** Every PR must confirm `just lint` passed and the image booted. Use `just boot-test` for automated pass/fail. No WIP PRs.

**Justfile integrity:** All maintenance tasks must be `just` recipes. No loose shell commands. If a task isn't covered by an existing recipe, add one before or alongside your change.

**Human maintainability:** Every agent action must be replicable by a human via the Justfile. No AI-optimized black boxes. Do not rename existing recipes without explicit human approval.

---

## Requirements

| Tool | Why | Install |
|---|---|---|
| `podman` (rootful + rootless) | BST container + export/boot | Pre-installed on Bluefin ✓ |
| `just` | All build/test commands | Pre-installed on Bluefin ✓ |
| `qemu` | VM boot | `brew install qemu` |
| `virtiofsd` | `just boot-fast` only | `rpm-ostree install virtiofsd` then reboot |
| `bcvk` | Ephemeral VM from container | Auto-installed by `just boot-fast` via cargo |
| ~100 GB disk, ~16 GB RAM | BST cache + parallel builds | — |

---

## Repo layout

| Path | Purpose |
|---|---|
| `elements/freedesktop-sdk.bst` | fdsdk junction — pinned to a release tag |
| `elements/gnome-build-meta.bst` | GBM junction — tracks `gnome-50` branch |
| `elements/bluefin/` | Bluefin-specific elements (~40 elements) |
| `elements/oci/` | OCI image assembly — layers + final image |
| `patches/freedesktop-sdk/` | Patches applied to fdsdk via `patch_queue` |
| `patches/gnome-build-meta/` | Patches applied to GBM via `patch_queue` |
| `patches/linux/` | Kernel patches (via fdsdk linux element) |
| `files/` | Static files installed by elements |
| `.github/skills/` | Repo-local agent skills |
| `Justfile` | All local dev commands — run `just --list` first |

---

## Dev loop

```bash
just validate                  # graph check — always run first (~5 min, no build)

export BUILD_SKIP_NVIDIA=1
just build default             # build image — warm cache: 2–5 min; cold: 60–90 min

just lint                      # bootc container lint — must pass before PR

just boot-test                 # automated smoke test — exits 0 on success
just boot-fast                 # interactive ephemeral VM via virtiofs (requires virtiofsd)

just show-me-the-future        # full loop: build → export → disk image → QEMU VM
```

First run is slow (cold BST cache). Subsequent runs are fast — BST caches by content hash.

---

## PR checklist

### All PRs

- [ ] `just validate` passes
- [ ] `just lint` passes on a built image
- [ ] `just boot-test` passes (or `just boot-fast` / `just boot-vm`)
- [ ] Commit trailer: `Assisted-by:` or `Signed-off-by:` — **not** `Co-authored-by:`
- [ ] `Closes #NNN` in the PR body

### Junction bumps (`gnome-build-meta.bst` or `freedesktop-sdk.bst`)

- [ ] Only junction `.bst` files changed — no `patches/` modifications in the same commit
- [ ] All existing patches in the relevant `patches/` directory still apply cleanly

> Junction-only bumps from `mergeraptor[bot]` are pre-approved once `validate` passes.

### Patch additions or removals (`patches/`)

- [ ] `Upstream-Status:` header: `Submitted` / `Accepted` / `Pending` / `Not-applicable`
- [ ] Upstream commit or PR linked if backporting
- [ ] Drop the patch if the fix is already upstream in the new junction ref
- [ ] Filenames numbered sequentially (alphabetical = apply order)
- [ ] Exit condition documented: "Drop when fdsdk ships X" or "Drop after GBM gnome-50 reaches Y"

### OCI image assembly (`elements/oci/`)

- [ ] `ldconfig -r /layer` present after `dconf update`, before `build-oci` — see [docs/oci-assembly.md](docs/oci-assembly.md)

### Element changes (`elements/bluefin/`)

- [ ] `mkdir -p` before any `ln -sf`
- [ ] Binary elements: `ref:` pinned to tag/commit, not a branch
- [ ] No `date`, `hostname`, `whoami`, `curl` in `install-commands`
- [ ] New systemd units enabled via BST install commands, not post-install scripts

---

## Community workflow

Issues flow: `status/discussing` → `status/approved` (maintainer writes acceptance criteria) → `queue/agent-ready` → `/claim` → implement → PR with `Closes #NNN` → merge queue.

**Actionadon bot:**

| Comment | Effect |
|---|---|
| `/claim` | Assigns you, adds `queue/claimed` |
| `/ready` | Moves approved+spec-complete issue into the queue (wranglers/maintainers only) |
| `/unclaim` | Returns to the queue |

**`kind:agent-donation` issues:** write the report as a comment, cite sources, close the issue. Do not open a PR.

**Hive:** Copy `files/hive/hive-project.yaml.example` to `/etc/hive/hive-project.yaml` and load `files/hive/agent-policies/` as per-agent CLAUDE.md overrides.

---

## Labels

| Label | Meaning |
|---|---|
| `status/discussing` | Not ready for the agent queue |
| `status/approved` | Approved; needs acceptance criteria before queue |
| `queue/agent-ready` | Scoped with acceptance criteria — claim it |
| `queue/claimed` | In active work |
| `agent/blocked` | Needs human input before work can continue |
| `hold` | Do not touch |
| `do-not-merge` | Do not merge or automate |
| `lgtm` | Maintainer approved |
| `lab:pass` | Lab validation passed; enables label-gated auto-merge |
| `kind:bug` / `kind:improvement` / `kind:tech-debt` / `kind:github-action` | Change type |
| `kind:agent-donation` | Investigation request — report comment, not code |
| `flow/project-report` / `flow/issue-review` / `flow/pr-review` | Hive scanner flow routing |
| `needs-human/agent-oops` | Agent error — do not remove; note what went wrong in your reply |

**Hive exempt** (do not touch): `hold`, `do-not-merge`, `status/discussing`, `status/approved`, `queue/claimed`, `agent/blocked`, `needs-human/agent-oops`, `duplicate`, `wontfix`, `stale`

---

## Patch management

Patches apply in **alphabetical filename order**. Numbers in filenames control application order.

When bumping a junction: verify every patch in the relevant `patches/` directory still applies; update or drop any that target a `ref:` no longer in the new junction. Kernel patches (`patches/linux/`) apply against kernel source — verify against the new kernel version too.

```
Add patch → Upstream-Status header → track upstream PR →
upstream merges → junction bump includes fix → drop patch
```

Every patch is maintenance debt. Drop as soon as it's upstream.

---

## CI

| Job | Triggers | What |
|---|---|---|
| `validate` | `pull_request` | `bst show` — graph + patch check (~5 min) |
| `build` | `merge_group`, `schedule`, `workflow_dispatch` | Full OCI build (~60–90 min) |
| `build-aarch64` | disabled | ARM64 — pending investigation |

Schedule: **13:00 UTC** daily (after GBM nightly at ~08:00 UTC). Never bypass the merge queue with `--admin`.

---

## What NOT to do

| Don't | Why |
|---|---|
| `rpm-ostree`, `pip install`, `apt-get` in elements | BST-only build; all deps from junctions |
| `$(date)`, `$(hostname)`, `$(curl ...)` in `install-commands` | Breaks reproducibility and BST caching |
| Patch junction files directly | Use `patch_queue` source in the junction `.bst` |
| Force-push to `main` | The merge queue owns merges |
| Close issues via API or comment | Use `Closes #NNN` in the PR body |
| Open a PR without running `just validate` | Wastes everyone's time |

---

## Useful BST commands

```bash
just validate                                           # check element graph
just bst build elements/bluefin/tailscale.bst          # build one element
just bst shell --build elements/bluefin/tailscale.bst  # sandbox shell
just bst show --deps all oci/bluefin.bst               # full dependency graph
```

---

## Links

- [BuildStream docs](https://docs.buildstream.build/)
- [freedesktop-sdk](https://gitlab.com/freedesktop-sdk/freedesktop-sdk)
- [gnome-build-meta](https://github.com/GNOME/gnome-build-meta) — branch `gnome-50`
- [Dakota issues](https://github.com/projectbluefin/dakota/issues)
- [Dakota board](https://github.com/orgs/projectbluefin/projects/3)
- [All Bluefin projects](https://github.com/orgs/projectbluefin/projects/2)
