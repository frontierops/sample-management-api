# Sample: provisioning semantic layers via the FrontierOps Management API

A working example of CI/CD-driven semantic-layer provisioning. Merging to
`main` applies every YAML file in this repo to the FrontierOps platform;
pull requests dry-run-validate them first. **No secrets are stored in this
repo** — the workflow authenticates with GitHub's OIDC identity against a
trust binding configured on the platform.

## Repository layout

```
orgs/
  test/                      ← the target organization's SLUG on FrontierOps
    connector-slug           ← the database connector SLUG in that org (one line)
    semantic-layers/
      analytics.yaml         ← the semantic layer's SLUG = the filename
```

Which org each file goes to is decided by the folder name. A folder for an
org outside the trust binding's grant fails with 403 for that file only.

**The connector is per-org.** Each customer org has its own database connector
(possibly a different vendor), so each `orgs/<slug>/` directory declares which
connector its semantic layers attach to via a one-line `connector` file
containing that connector's slug. The connector must already exist in the org
and be a database-type connector; create it once in the dashboard
(Connectors → Add) before provisioning layers that reference it.

## One-time setup

1. **Platform** (an org admin, on the parent org): Settings → Management API
   → New Trust Binding. Recommended pair for this repo:
   - *Apply*: branch ref `refs/heads/main`, scopes semantic layers read + write
   - *Validate*: no branch ref, scope semantic layers read only

   The token exchange picks the right one automatically (most specific wins).
2. **This repo**: nothing. The workflow file's `id-token: write` permission is
   the only "configuration" — no secrets, no variables, no GitHub Apps.
3. Adjust `env.CONNECTOR` in the workflow to the database connector slug that
   exists in the target org (the semantic layer attaches to it).

## What happens on each event

| Event | Job | Effect |
|---|---|---|
| Pull request | `validate` | Each YAML dry-runs against `POST …/validate`; broken YAML fails the check with errors in the log. Nothing is written. |
| Push to `main` | `provision` | Idempotent `PUT …/by-slug/{slug}` per file: unchanged YAML = no-op (version stays), changed = version bump, new file = created. |

Deletes are deliberate: removing a YAML file does **not** delete the layer.

Tip: mark the `validate` job as a required status check in branch protection
so an invalid semantic layer can never reach `main`.

## Troubleshooting

- `401 invalid_grant` — no binding matches (check the repo is `owner/repo`
  exactly, branch pin, binding enabled) or the OIDC token was already used.
- `403` on a PUT — that org isn't in the binding's grant, the binding was
  disabled, or the token lacks write scope.
- `503 jwks_unavailable` — transient GitHub keys outage; re-run the job.
- Every applied change appears in Settings → Management API → the binding's
  Activity panel, with the GitHub run id and author.
