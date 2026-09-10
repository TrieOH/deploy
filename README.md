# deploy

Production runtime for TrieOH's self-hosted stack. **TheTree** (source + CI)
builds, checks, and publishes images — **and deploys them**: pushing a release
tag in TheTree makes the pipeline update this repo automatically.

**Humans don't deploy from here anymore — TheTree tags deploy.** Any dev with
push access to TheTree can ship a release; the pipeline is the quality gate
(all checks run before anything is published) and the only writer of this
repo's ledger. Humans keep read access and rollback.

One folder per project at the root. Currently: **`thetree/`**. The next
project adds its own folder.

## Layout (`thetree/`)

```
thetree/
├── compose.yml          # prod stack: postgres, rustfs, identityx, univents, payssage, informd
├── .env.example         # shared: POSTGRES_*, RUSTFS_* — blank values (committed)
├── .env                 # shared real values (gitignored — server only)
├── .identityx.env(.example)   # per-service env, one per backend
├── .univents.env(.example)
├── .payssage.env(.example)
└── .informd.env(.example)
```

## Deploy (automated)

A release in TheTree (`<artifact>/v<semver>` tag) runs the full pipeline:
checks → build once → publish immutable image → **the pipeline commits here**,
swapping the image line to the new digest (`release: <svc> vX.Y.Z`). On every
commit, a workflow on this repo SSHes to the server and runs:

```bash
git pull --ff-only && docker compose pull && docker compose up -d --no-deps
```

Idempotent — compose only recreates containers whose image digest actually
changed (`name: thetree` must never change). One rollout at a time
(concurrency-gated).

## Rollback

```bash
git revert <ledger-commit>   # or checkout the previous compose.yml
git push                     # the workflow redeploys the previous digest
```

Frontends roll back in Cloudflare: `wrangler versions deploy --version-id
<previous>` (deployment history keeps every version).

## Env handling

- Real env files (`*.env`) are gitignored and exist only on the server.
- Templates (`.*.env.example`) are committed with **blank** values and
  document prod state, not dev.
- **Never commit a snapshot.** `docker compose config` inlines every
  resolved secret into its output. If you need a baseline, keep it in
  `~/backups/` — never in this repo. (We learned this the hard way; the
  first committed snapshot leaked all prod secrets and had to be scrubbed
  from history.)

## Versions

`compose.yml` pins **digests** — `image: git.trieoh.com/trieoh/<svc>@sha256:…
# vX.Y.Z` (the human tag lives in a comment, never in the image reference).
**Version changes are made by TheTree's release pipeline, never by hand.**
Git history is the version ledger. To see what's deployed and when:
`git log --oneline -- thetree/compose.yml`.

Rollout commits read `release: <svc> vX.Y.Z` (or `vX.Y.Z-hotfix.N` for the
express lane). The current pin (as of writing): payssage digest-pinned;
identityx/univents still tag-pinned — they flip to digests on their next
releases; informd is unused in prod.

## Add a service

1. Add the service to `compose.yml` (`image`, `container_name`, `env_file`,
   `networks` — keep the existing patterns). Use the tag form
   (`git.trieoh.com/trieoh/<svc>:v0.0.1`); the pipeline digest-pins it on the
   first release.
2. Create `.<svc>.env.example` from the live server file, values blanked.
3. Tag a release in TheTree + smoke-test.

## Server

- Host `trieoh@main`; project dir `~/deploy/thetree`.
- The nightly dind-prune and forgejo-restart crons live in the server's
  crontab (not in git) — when rebuilding the box, re-add them.
- Infra services (caddy, forgejo, mox, observability, beszel, ntfy) live in
  `TrieOH/infra`, not here.
