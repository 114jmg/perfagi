# PerfAGI Source Fork Guide

This repository is a fork of [vxcontrol/pentagi](https://github.com/vxcontrol/pentagi) (MIT License) retooled into a multi-agent performance-testing platform. The upstream LICENSE stays at the repository root.

- Upstream policy: **follow** — merge each upstream release instead of hard-forking away.
- Rationale: upstream is active (32k stars, weekly cadence); security fixes (CVE-2026-14784) and new LLM support (DeepSeek V4, provider failover) are worth taking for free.
- Conflict expectation: confined to the registration-point files listed in `PERF-FORK-CHANGES.md`; each merge should cost under half a day. AI-assisted conflict resolution works well: hand the conflicting hunk plus both sides' intent to an AI, then review its choice.

## Branch model

| Branch | Role |
|---|---|
| `origin/main` | Upstream mirror, fetch-only, never pushed to |
| `perf-fork` | Main development branch, cut from `v2.2.0` |

## Fork discipline (what keeps merges cheap)

1. **Prefer new files**: the new agent (`pkg/providers/performer.go`), new tools (`pkg/tools/perf*.go`), new frontend pages all live in new files — nothing gets pasted into upstream files.
2. **Upstream files take registrations only**: when an upstream file must change, the change is an added line (a tool registered in `tools/registry.go`, a PromptType in `templates.go`), never a refactor, move or rename of upstream code.
3. **Ledger rule**: every edit to an upstream file is recorded in `PERF-FORK-CHANGES.md` (file, intent, expected conflict point); merges are reconciled against that list.
4. **Commit granularity**: one feature per commit, for cherry-picks and rollbacks.
5. **Pentest capability stays**: this is a dual-purpose platform; perf becomes a first-class citizen, nothing pentest-shaped gets deleted. A deletion needs a recorded reason in the ledger first.

## Routine upstream sync (once per upstream release)

```bash
HTTPS_PROXY=http://127.0.0.1:7897 git fetch origin --tags
git log perf-fork..origin/main --oneline          # what moved upstream
git switch perf-fork
git merge v2.3.0                                  # follow release tags, not loose main commits
# Resolve conflicts against PERF-FORK-CHANGES.md: check each listed registration point
# still carries both upstream's additions and ours.
# Verify:
docker build --build-arg GOPROXY=https://goproxy.cn,direct \
  --build-arg APT_MIRROR=mirrors.ustc.edu.cn \
  --build-arg APK_MIRROR=mirrors.tuna.tsinghua.edu.cn \
  --build-arg PACKAGE_REV=$(git rev-parse --short HEAD) \
  --tag perfagi/dev:merge-test .
# On success: docker save/load to the VM, smoke-test, then tag.
```

## Building (local Mac, Docker Desktop ≥ 8 GB)

- The stock Debian/Alpine CDN edges (Fastly) intermittently 502 from this network, and proxy.golang.org is blocked outright; all three are bypassed with mirror build args — everything else (npm registry) is directly reachable.
- If a proxy ever becomes necessary: Clash on this Mac only accepts loopback connections (macOS Local Network privacy blocks `host.docker.internal:7897` from containers, and `--network=host` on Docker Desktop reaches the Linux VM, not the Mac), so grant Clash Local Network access in System Settings first.
- The Vite frontend build wants a 4 GB heap; give Docker Desktop ≥ 6 GB.
- The VM (3.8 GB RAM) never builds — it receives images: `docker save perfagi/dev:x | gzip | ssh dsh-vm 'gunzip | docker load'`.
- Local Go work (outside Docker) needs once per fresh clone: `go generate ./cmd/installer/files/` (see CLAUDE.md).

```bash
REV=$(git rev-parse --short HEAD)
docker build \
  --build-arg GOPROXY=https://goproxy.cn,direct \
  --build-arg APT_MIRROR=mirrors.ustc.edu.cn \
  --build-arg APK_MIRROR=mirrors.tuna.tsinghua.edu.cn \
  --build-arg PACKAGE_REV=$REV \
  --tag perfagi/dev:2.2.0-base .
```

Pass `PACKAGE_REV` and nothing else: `PACKAGE_VER` must stay empty for non-release builds — a word like `ce` makes the label read `ce-ce` while the binary reports `ce` (the version rule in the Dockerfile tail explains why).

## Deployment upgrade notes (2.1-era image → 2.2+)

- Eight DB migrations run automatically; every user must sign in again.
- Custom prompts: the old per-engine variables (`GoogleToolName` and friends) are gone — use `WebSearchToolName` (the perf prompts already comply, verified).
- `DOCKER_INSIDE=false` is the 2.2 default; the VM already matches.
- On the VM compose, switch the image field from `vxcontrol/pentagi:latest` to `perfagi/dev:<tag>` and `docker compose up -d pentagi`.

## Backup remote (strongly recommended, against Mac disk loss)

```bash
# After creating a private GitHub repository:
git remote add fork git@github.com:<account>/perfagi.git
git push fork perf-fork
```
