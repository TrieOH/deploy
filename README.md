# deploy

Production runtime for TrieOH's self-hosted stack. **TheTree** (source + CI)
builds and publishes images; **this repo** runs them.

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

## Deploy

```bash
cd ~/deploy/thetree
docker compose up -d --no-deps
```

Idempotent — compose adopts existing containers (container names, volumes and
networks are pinned and unchanged; `name: thetree` must never change).

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

Images are pinned to what the server runs: `identityx:v0.35.3`,
`univents:v0.10.4`, `payssage:v0.7.4`, `informd:latest` (informd was never
pinned — pin it on its next release). `:latest` is an explicit escape hatch.

**Bumping a version** = edit the tag in `compose.yml`, commit, deploy. Git
history is the version ledger. **Rollback** = `git checkout <prev> -- compose.yml`
and redeploy.

## Add a service

1. Add the service to `compose.yml` (`image`, `container_name`, `env_file`,
   `networks` — keep the existing patterns).
2. Create `.<svc>.env.example` from the live server file, values blanked.
3. Deploy + smoke-test.
4. TheTree side already produces the image (Dockerfile + `publish.yml`).

## Server

- Host `trieoh@main`; project dir `~/deploy/thetree`.
- The nightly dind-prune and forgejo-restart crons live in the server's
  crontab (not in git) — when rebuilding the box, re-add them.
- Infra services (caddy, forgejo, mox, observability, beszel, ntfy) live in
  `TrieOH/infra`, not here.
