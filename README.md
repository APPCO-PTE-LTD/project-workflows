# project-workflows

Reusable GitHub Actions workflows that keep our **GitHub Projects** boards in step with issues and
pull requests, so nobody has to drag cards.

Public on purpose: on the free plan a **private** repository's reusable workflows cannot be called
from other repositories. Nothing here is sensitive — workflow YAML only, no secrets, no company
data. The design rationale lives in our internal SDLC, §7.8 Automation (organization members only).

## What is here

| Workflow                                                               | Trigger in the caller        | What it does                                          |
| ---------------------------------------------------------------------- | ---------------------------- | ----------------------------------------------------- |
| [`add-to-project.yml`](.github/workflows/add-to-project.yml)           | issue or PR opened           | Adds the item to the product's Project                |
| [`set-issue-in-review.yml`](.github/workflows/set-issue-in-review.yml) | PR opened / ready for review | Moves the issue the PR closes to `Status = In Review` |
| same, with `status: Testing`                                           | PR merged                    | Moves the issue the PR closes to `Status = Testing`   |

Everything else on the board is handled by the Project's **built-in** workflows — item added, item
reopened, item closed, PR merged, changes requested, auto-archive — which need no code. These
exist only because the built-ins cannot do them:

- **Auto-add** is limited to **one repository per project** on the free plan (5 on Team, 20 on
  Enterprise), and a product has around four repositories.
- **"PR opened → issue In Review"** has no built-in equivalent, and `In Review` is the status that
  goes stale fastest.
- **"PR merged → issue Testing"**: the built-in *PR merged* moves only the PR's own card.

Per §7.8, only unambiguous transitions are automated. `Ready for Deployment` and `Done` stay manual —
automating a judgement produces a board that lies. QA closes the issue when it passes testing, and
*item closed* sets `Done`.

## Using it in a product repository

1. Copy [`examples/project-sync.yml`](examples/project-sync.yml) to
   `.github/workflows/project-sync.yml` and replace `<N>` with the Project number.
2. Add the `PROJECT_AUTOMATION_TOKEN` secret (below).
3. Turn **off** Settings → General → *Auto-close issues with merged linked pull requests*. `nightly`
   is the default branch, so otherwise merging a PR closes its issue and the board marks it `Done`
   before QA has tested it.
4. Open a test issue. It should appear on the board at `Status = Backlog`.

Calls are pinned to `@v1`. §6.11 forbids floating refs — never change these to `@main`.

## The token

`GITHUB_TOKEN` **cannot** write to organization Projects, so these workflows need their own.

Create a **fine-grained personal access token**:

| Setting                  | Value                                                        |
| ------------------------ | ------------------------------------------------------------ |
| Resource owner           | `APPCO-PTE-LTD`                                              |
| Organization permissions | `Projects` → **Read and write**                              |
| Repository permissions   | `Issues` → Read · `Pull requests` → Read · `Metadata` → Read |
| Expiry                   | 1 year — put the rotation date in a calendar                 |

Store it as a **repository** secret named `PROJECT_AUTOMATION_TOKEN` in each repository that syncs
to a board.

> Once the organization is on **GitHub Team**, move it to a single organization secret restricted to
> the product repositories, and delete the per-repository copies. Organization secrets are not
> readable by private repositories on the free plan, which is the only reason it is duplicated.

## Releasing a change

Callers pin `@v1`, so a change is not live until the tag moves:

```bash
git tag -f v1 && git push -f origin v1
```

Moving `v1` changes behaviour for every repository at once. Read the diff twice.
