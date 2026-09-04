# cgm-remote-monitor — homelab fork (Nightscout)

This is Michael's fork of [Nightscout/cgm-remote-monitor](https://github.com/nightscout/cgm-remote-monitor).

> ## ⚠️ OUT OF THE DEPLOY PATH SINCE 2026-09-04 — READ THIS FIRST
>
> **This repo no longer builds the image that runs
> https://nightscout.familie-kroes.nl.** That stack
> (`~/repos/portainer-stacks`, `stacks/nightscout/`) now runs the **official
> upstream release** `nightscout/cgm-remote-monitor:15.0.8` straight from Docker
> Hub. The "Build + deploy procedure" below is kept as reference for how to build
> a local image *if one is ever needed again*, but it is **not** how Nightscout is
> deployed today, and running it will not change what is live.
>
> Why: the fork's app-code divergence from upstream reached **zero**.
> - "Fixed the OpenAPS pill and forecast lines from not showing when unexpected
>   data is present in lastEnacted" (`d2450fcb`) was **merged upstream** as
>   `aabaebf6` via PR nightscout/cgm-remote-monitor#8324 — tests and fixtures
>   included — then hardened further by `41dddbaa` + `d198a3cc`. Upstream's
>   version is a superset: it guards `duration` as well as `rate`, and omits the
>   Temp Basal segment entirely rather than rendering a malformed one. First
>   release containing it: **15.0.8** (2026-09-04).
> - "(Removed global open warning)" (`deb93071`) stubbed out the boot warning
>   behind `authDefaultRoles == 'readable'`. Dead since 2026-08-06, when the
>   instance's `.env` was set to `AUTH_DEFAULT_ROLES=denied` — that branch cannot
>   fire, so the patch guards nothing.
>
> Only `CLAUDE.md` (this file) is still fork-only, and it is not app code.
>
> **`personal-fixes` is therefore ~318 commits behind `upstream/master` and is
> deliberately NOT being kept in sync.** Don't "fix" that drift — there is nothing
> on it that upstream lacks, and merging 318 commits into a branch nothing builds
> from is pure cost. If you want the branch tidy, the honest end state is to reset
> it to `upstream/master`, not to merge.
>
> **What this repo is FOR now:** developing the next patch and sending it upstream.
> The OpenAPS fix is the worked example of why — upstream-first took one PR and
> then permanently removed a build step, a `pull_policy: never` workaround, and a
> whole class of "is the deployed image current?" question. Start from
> `upstream/master`, branch, PR. Reach for a local image only as a stopgap while a
> PR is in flight, and if you do, say so in the compose file's comment.

Until 2026-09-04 this repo built the custom Docker image that ran the family's
Nightscout instance at https://nightscout.familie-kroes.nl (deployed from
`~/repos/portainer-stacks`, stack `stacks/nightscout/`).

## Remotes

- `origin` — `git@github.com:yodax/cgm-remote-monitor.git` (this fork, push access)
- `upstream` — `git@github.com:nightscout/cgm-remote-monitor.git` (read-only, upstream project)

## Branch: `personal-fixes`

**`personal-fixes` is the one permanent branch for all custom work.** Make fixes
directly on it, commit, then build+deploy (procedure below). There is no more
per-change branching.

This replaces an old convention (retired 2026-07-21) of cutting a new dated branch
(`2023-02-05`, `2024-12-05`, `2026-05-02`, ...) each time upstream was synced. Those
old branches still exist on `origin` for history but are dead — don't build from them.

`personal-fixes` was cut from `upstream/master` plus two standing patches that predate
this restructure (now just ordinary commits in its history, no special handling needed):
- "Removed global open warning"
- "Fixed the OpenAPS pill and forecast lines from not showing when unexpected data is
  present in lastEnacted"

### Pulling in upstream updates

Periodically (not on every fix) bring upstream changes in:

```bash
git fetch upstream
git checkout personal-fixes
git merge upstream/master   # resolve conflicts if any, then commit
git push origin personal-fixes
```

Use a merge, not a rebase — this branch is already pushed and long-lived; rebasing
would force-push and rewrite shared history.

## Build + deploy procedure (HISTORICAL — see the banner at the top)

**This is no longer how Nightscout is deployed.** Kept only so a future stopgap
local build does not have to be reinvented. The live stack pulls
`nightscout/cgm-remote-monitor:<release>` from Docker Hub.

There is no CI for this — `.github/workflows/main.yml`'s publish jobs are gated to
`github.repository_owner == 'nightscout'`, so they never fire on this fork. Building
is always this manual (but scriptable) process. **This devbox has no Docker daemon**
(and the Dockerfile's node version wouldn't match a local install anyway), so the
image is always built on ct100, the actual Docker host.

Image tag convention: the short git commit SHA of `personal-fixes` at build time
(e.g. `nightscout:a1b2c3d`) — guarantees each image is traceable to an exact commit,
even across multiple same-day builds.

```bash
# 1. From this repo, on the personal-fixes branch, after committing your fix:
cd ~/repos/cgm-remote-monitor
git checkout personal-fixes
SHA=$(git rev-parse --short HEAD)
git archive --format=tar --output=/tmp/nightscout-$SHA.tar HEAD

# 2. Ship the source to ct100 and build there (ct100 has no git credentials for
#    this repo, so we transfer a tarball rather than cloning on-box):
scp /tmp/nightscout-$SHA.tar pve:/tmp/nightscout-$SHA.tar
ssh pve "pct push 100 /tmp/nightscout-$SHA.tar /tmp/nightscout-$SHA.tar && rm /tmp/nightscout-$SHA.tar"
ssh pve "pct exec 100 -- bash -c '
  mkdir -p /tmp/nightscout-build && \
  tar -xf /tmp/nightscout-$SHA.tar -C /tmp/nightscout-build && \
  cd /tmp/nightscout-build && \
  docker build -t nightscout:$SHA .
'"

# 3. Clean up build artifacts on ct100 (the image itself stays in ct100's local
#    Docker cache — it's never pushed to a registry):
ssh pve "pct exec 100 -- rm -rf /tmp/nightscout-build /tmp/nightscout-$SHA.tar"
rm /tmp/nightscout-$SHA.tar

# 4. (Recommended) smoke-test the image boots before wiring it into the live stack —
#    use a throwaway container with a bogus MONGO_CONNECTION, confirm it gets past
#    checkNodeVersion/checkSettings and starts retrying the Mongo connection (that's
#    success — it means the app itself booted fine), then remove it:
ssh pve "pct exec 100 -- docker run -d --name ns-smoketest -e API_SECRET=smoketestsmoketest \
  -e MONGO_CONNECTION=mongodb://127.0.0.1:27017/smoketest -e INSECURE_USE_HTTP=true nightscout:$SHA"
ssh pve "pct exec 100 -- docker logs ns-smoketest" # expect checkNodeVersion/checkSettings to pass
ssh pve "pct exec 100 -- docker rm -f ns-smoketest"

# 5. Point the live stack at the new image (in ~/repos/portainer-stacks):
cd ~/repos/portainer-stacks
# edit stacks/nightscout/docker-compose.yml — bump the image tag and provenance
# comment to nightscout:$SHA
git add stacks/nightscout/docker-compose.yml
git commit -m "Update nightscout to <describe the fix>"
git push

# 6. Deploy via the sanctioned path (never docker compose up by hand — see
#    portainer-stacks CLAUDE.md hard rule 3 / .claude/skills/deploy):
ssh pve "pct exec 100 -- bash -c 'cd /opt/stacks && git fetch -q && git log HEAD..origin/master --oneline'"
# ^ confirm the pull range only touches stacks/nightscout — if other stacks changed
#   upstream in portainer-stacks meanwhile, they'll deploy too; that's expected
#   update.sh behavior, not a bug, but worth noticing before proceeding.
ssh pve "pct exec 100 -- bash -c 'cd /opt/stacks && bash update.sh'"

# 7. Verify:
ssh pve "pct exec 100 -- docker ps --filter name=nightscout --format '{{.Names}}\t{{.Image}}\t{{.Status}}'"
ssh pve "pct exec 100 -- docker logs --tail 30 nightscout"
curl -sI https://nightscout.familie-kroes.nl
```

### Why `update.sh` works here despite the image having no registry

`nightscout:<sha>` is a bare, unqualified, locally-built tag — `docker compose pull`
will always fail for it ("pull access denied", since Docker defaults to looking on
Docker Hub). `update.sh` in `portainer-stacks` tolerates this (treats a failed pull as
a warning and recreates with the existing local image) — but that only works if the
image already exists in ct100's local Docker cache *before* `update.sh` runs. That's
why step 2 (build) must happen before step 6 (deploy): the image has to be sitting on
ct100 already, or the deploy will recreate a container against an image that isn't there.

### A gotcha to know about

`update.sh` starts with `git pull` on itself — if you're ever also changing
`update.sh`'s own content in the same push as a stack change, run `update.sh` an
extra time afterward: bash can misbehave reading a script file that changes out
from under it mid-execution (self-modifying script hazard). This doesn't apply to
routine nightscout deploys, which only touch `stacks/nightscout/docker-compose.yml`.

## Cross-references

- `~/repos/portainer-stacks/CLAUDE.md` — the homelab's overall rules (git-commit
  discipline, confirmation requirements for risky changes, etc.) apply to the deploy
  half of this workflow too.
- `~/repos/portainer-stacks/.claude/skills/deploy/` — the general `update.sh` deploy
  skill this procedure builds on.
- `~/repos/portainer-stacks/stacks/nightscout/docker-compose.yml` — the live stack
  definition (mongodb + nightscout + Traefik routing).
