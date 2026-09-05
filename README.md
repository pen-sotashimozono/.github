# pen-sotashimozono/.github

Organization-level shared configuration for **pen-sotashimozono**.

## Reusable workflows

Shared GitHub Actions live in [`.github/workflows/`](.github/workflows) as
[reusable workflows](https://docs.github.com/actions/using-workflows/reusing-workflows)
(`on: workflow_call`). A paper repository keeps a thin caller per workflow that
owns the `on:` triggers and delegates the body here, so the pipeline is
maintained in one place.

| Reusable workflow | Purpose | Caller file |
| --- | --- | --- |
| `paper-build.yml` | Build every document in `docs.toml`; on a PR, check that exactly the affected documents bumped | `latex-ci.yml` |
| `paper-release.yml` | Build and publish one document as `v<version>-<document>` with PDF, latexdiff PDF and arXiv bundle | `Release.yml` |
| `paper-auto-release.yml` | After a merge, dispatch a release for every document whose version has no tag yet | `AutoRelease.yml` |
| `paper-version-check.yml` | Build-free check that each version in `docs.toml` moved at most one semver step | `VersionCheck.yml` |
| `paper-verify-references.yml` | Resolve every DOI / arXiv id in `references.bib` against Crossref and arXiv | `VerifyReferences.yml` |
| `latex-ci.yml` | **Deprecated.** Single-root build on the `rosina` runner, no `docs.toml`. Kept for `MasterThesis` only | — |

### What stays in the paper repository

`.github/scripts/` is **not** shared. `bump.sh`, `diff.sh`, `refs_sync.sh` and
`fetch_sources.sh` are run by hand and import `docs.py`, `closure.py` and
`bibentries.awk` from that same directory, so moving half of them here would
break the local half. Every reusable workflow above checks out the caller and
runs the caller's copy.

`docs.toml`, `.latexmkrc`, `.github/CHANGELOG.md` and the `.tex` roots are
likewise per-repository.

### Pinning

Callers reference `@v1`, a moving tag on this repository, not `@main`: a bad
push to `main` would otherwise break every paper repository at once. To roll a
change out, merge to `main`, point one repository at `@main` to try it, then
move the tag:

```sh
git tag -f v1 main && git push -f origin v1
```

### The shelf

`paper-release.yml` ends with a `shelve` job that mirrors the PDF it just
published into [`paper-shelf`](https://github.com/pen-sotashimozono/paper-shelf),
so every document in the organisation is readable from one directory. It is
**inert until the deploy key exists**: with no key the job reports that and
succeeds. Callers pass the secret anyway, so switching it on touches no
repository.

To switch it on:

```sh
ssh-keygen -t ed25519 -N "" -C "paper-shelf write" -f ~/.ssh/shelf
gh repo deploy-key add ~/.ssh/shelf.pub -R pen-sotashimozono/paper-shelf --allow-write -t "release shelving"
gh secret set SHELF_DEPLOY_KEY --org pen-sotashimozono --visibility all < ~/.ssh/shelf
rm ~/.ssh/shelf ~/.ssh/shelf.pub
```

A deploy key rather than a PAT: it writes to the shelf and to nothing else, and
it does not expire. `shelf-repo` can be set to an empty string by a caller that
should never shelve.

### Caller templates

Copy these into `<repo>/.github/workflows/`. The file names matter:
`paper-auto-release.yml` dispatches the caller's `Release.yml` by name.

```yaml
# <repo>/.github/workflows/latex-ci.yml
name: LaTeX CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
permissions:
  contents: read
jobs:
  build:
    uses: pen-sotashimozono/.github/.github/workflows/paper-build.yml@v1
```

```yaml
# <repo>/.github/workflows/Release.yml
name: Release
on:
  push:
    tags: ["v[0-9]*.[0-9]*.[0-9]*-*"]
  workflow_dispatch:
    inputs:
      document:
        description: "Document id as it appears in docs.toml (e.g. main)"
        required: true
      version:
        description: "Version without 'v' prefix (e.g. 1.2.3)"
        required: true
permissions:
  contents: write
jobs:
  release:
    uses: pen-sotashimozono/.github/.github/workflows/paper-release.yml@v1
    permissions:
      contents: write
    with:
      # Empty on a tag push, where the pair is read back out of the tag name.
      document: ${{ inputs.document || '' }}
      version: ${{ inputs.version || '' }}
    secrets:
      # Empty until the shelf deploy key exists, and the shelve job then says so
      # and does nothing. Passing it from the start is what makes switching
      # shelving on a matter of creating one secret and editing no repository.
      shelf-deploy-key: ${{ secrets.SHELF_DEPLOY_KEY }}
```

```yaml
# <repo>/.github/workflows/AutoRelease.yml
name: Auto Release
on:
  pull_request:
    types: [closed]
    branches: [main]
permissions:
  contents: write
  actions: write
jobs:
  release:
    if: github.event.pull_request.merged == true
    uses: pen-sotashimozono/.github/.github/workflows/paper-auto-release.yml@v1
    permissions:
      contents: write
      actions: write
```

```yaml
# <repo>/.github/workflows/VersionCheck.yml
name: Version Check
on:
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  check:
    uses: pen-sotashimozono/.github/.github/workflows/paper-version-check.yml@v1
```

```yaml
# <repo>/.github/workflows/VerifyReferences.yml
name: Verify references
on:
  pull_request:
    paths:
      - "references.bib"
      - ".github/workflows/VerifyReferences.yml"
  push:
    branches: [main]
    paths:
      - "references.bib"
  workflow_dispatch:
permissions:
  contents: read
jobs:
  verify:
    uses: pen-sotashimozono/.github/.github/workflows/paper-verify-references.yml@v1
    with:
      contact-email: you@example.org
```
