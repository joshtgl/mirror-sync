# mirror-sync

Scheduled GitHub Actions workflow that mirrors external git repositories into
GitHub. Every entry in [`mirrors.json`](mirrors.json) is synced every 6 hours
(and on demand) by [`.github/workflows/mirror.yml`](.github/workflows/mirror.yml).

Only branches and tags are pushed (not `refs/pull/*`, which GitHub rejects).
`--prune` deletes branches/tags that were removed upstream, and force-pushes
upstream are reproduced as-is — treat mirror repos as read-only.

## mirrors.json

```json
[
  {
    "name": "forgejo-helm",
    "source": "https://code.forgejo.org/forgejo-helm/forgejo-helm.git",
    "target": "forgejo-helm-mirror/forgejo-helm",
    "token_secret": "FORGEJO_HELM_MIRROR_TOKEN"
  }
]
```

| Field     | Required | Description |
|-----------|----------|-------------|
| `name`    | yes      | Unique short name (used for job names, concurrency, and the `only` input). |
| `source`  | yes      | Public clone URL of the upstream repository. |
| `target`  | yes      | GitHub `owner/repo` to push to. |
| `refs`    | no       | Ref patterns to mirror. Default `["refs/heads/*", "refs/tags/*"]`. |
| `enabled` | no       | Set to `false` to pause a mirror without removing it. |
| `token_secret` | no  | Name of the Actions secret holding the push token. Default `MIRROR_TOKEN`. |

## One-time setup

Fine-grained tokens belong to a single resource owner (user or org), so use
one token per owner and point each mirror at it with `token_secret`.

1. Create a **fine-grained personal access token** with **Resource owner** set
   to the mirrors' org (e.g. `forgejo-helm-mirror`), access to **All
   repositories**, and these repository permissions:
   - **Contents:** Read and write
   - **Workflows:** Read and write (required if upstream contains `.github/workflows/*`)
2. Add it to this repository as an Actions secret, e.g.
   **`FORGEJO_HELM_MIRROR_TOKEN`**. If the org requires approval for
   fine-grained tokens, approve it under the org's Settings → Personal access
   tokens → Pending requests.

## Adding a mirror

1. Create an **empty** GitHub repository (no README/license/.gitignore).
2. In that repo, **disable Actions** (Settings → Actions → General → Disable actions)
   so upstream CI files never run there. Optionally disable Issues/PRs and note
   "read-only mirror" in the description.
3. Make sure a token can push to it: either it's in an org whose token covers
   all repositories, or create a new token/secret for its owner.
4. Add an entry to `mirrors.json` (with `token_secret`) and push.
5. Run the workflow manually (Actions → mirror → Run workflow) with `only` set
   to the new mirror's name to do the initial sync.

## Running manually

```bash
gh workflow run mirror.yml                    # all mirrors
gh workflow run mirror.yml -f only=forgejo-helm
```

## Verifying

```bash
diff <(git ls-remote https://code.forgejo.org/forgejo-helm/forgejo-helm.git | grep -v refs/pull/ | grep -v HEAD) \
     <(git ls-remote https://github.com/forgejo-helm-mirror/forgejo-helm.git | grep -v HEAD)
```
