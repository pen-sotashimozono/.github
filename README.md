# pen-sotashimozono/.github

Organization-level shared configuration for **pen-sotashimozono**.

## Reusable workflows

Shared GitHub Actions live in [`.github/workflows/`](.github/workflows) as
[reusable workflows](https://docs.github.com/actions/using-workflows/reusing-workflows)
(`on: workflow_call`). Paper repositories keep a thin caller that delegates the
actual logic here, so CI is maintained in **one place** and runs on the
self-hosted `rosina` runner.

| Reusable workflow | Purpose |
| --- | --- |
| `latex-ci.yml` | Build the paper PDF on push / PR |

### How a repo calls it

```yaml
# <repo>/.github/workflows/latex-ci.yml
name: LaTeX CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
jobs:
  build:
    uses: pen-sotashimozono/.github/.github/workflows/latex-ci.yml@main
```
