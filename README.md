# TheDesignAgent design review

A design review gate for pull requests. On each preview deploy, it finds the UI pages a pull request changed, reviews them with [TheDesignAgent](https://www.thedesignagent.ai), posts the results as one PR comment that updates on every push, and fails the check if a page scores below your threshold.

- **UX review:** job fit, UX heuristics, cognitive load and pattern conformance, scored 0–10 with findings and fixes.
- **Visual review:** brand compliance, visual hierarchy and aesthetics, scored 0–10 from a real screenshot of the preview.
- **Project memory:** reviews use the same project context as your coding agents, from the `.thedesignagent` file in your repo.

## Setup

1. Get an API key at [thedesignagent.ai/dashboard/api-keys](https://www.thedesignagent.ai/dashboard/api-keys) and add it as a repository secret named `THEDESIGNAGENT_API_KEY`.
2. If your Vercel previews are protected, create a **Protection Bypass for Automation** secret in Vercel (Project → Settings → Deployment Protection) and add it as `VERCEL_AUTOMATION_BYPASS_SECRET`.
3. Add a workflow. For Vercel previews, copy [`examples/vercel-preview.yml`](examples/vercel-preview.yml) to `.github/workflows/design-review.yml`:

```yaml
name: Design review

on:
  deployment_status:

jobs:
  design-review:
    if: github.event.deployment_status.state == 'success' && !contains(github.event.deployment_status.environment, 'Production')
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.event.deployment.sha }}
          fetch-depth: 0
      - uses: CMBurnett/thedesignagent-action@v1
        with:
          api-key: ${{ secrets.THEDESIGNAGENT_API_KEY }}
          vercel-bypass-secret: ${{ secrets.VERCEL_AUTOMATION_BYPASS_SECRET }}
          threshold: 6
```

Vercel posts a deployment status to GitHub when a preview is ready, so the review runs against the live preview of that commit, and the comment lands on its pull request. To make it a gate, mark the **design-review** check as required in your branch protection rules.

Not on Vercel? Use [`examples/pull-request.yml`](examples/pull-request.yml) with a `base-url` that serves the branch, or any host that posts GitHub deployment statuses.

## Which pages get reviewed

The action diffs the pull request against its base branch and maps changed files to [Next.js App Router](https://nextjs.org/docs/app) pages: a change to `app/orders/_components/table.tsx` reviews `/orders`. Dynamic routes such as `app/orders/[id]/page.tsx` have no URL to visit, so list a concrete page for them in `.thedesignagent`:

```json
{
  "project_id": "proj_...",
  "check": {
    "threshold": 6,
    "pages": [
      { "path": "/orders/123", "code": "app/orders/[id]/page.tsx", "task": "Review an order and decide whether to refund it" }
    ]
  }
}
```

A `task` that describes the user's job makes the UX review sharper. Pages behind login need a saved session: set `THEDESIGNAGENT_AUTH_SEED` as described in the [`@thedesignagent/mcp` README](https://www.npmjs.com/package/@thedesignagent/mcp).

To see what a run would cover without reviewing anything, run `npx -y --package=@thedesignagent/mcp thedesignagent check --changed --base-url <preview> --dry-run` locally.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `api-key` | required | Your TheDesignAgent API key |
| `base-url` | preview URL from the deployment event | Where the pages live |
| `checks` | `ux,visual` | Reviews to run on each page |
| `threshold` | none | Fail if any page scores below this (0–10). Without it the action reports but never fails |
| `vercel-bypass-secret` | none | For protected Vercel previews. Sent only to the preview's own domain |
| `comment` | `true` | Post and update one PR comment |
| `base` | the PR's base branch | Git ref to diff against |
| `version` | `0.4` | Version of `@thedesignagent/mcp` to run |

## Exit codes

- `0`: all reviewed pages passed, or no UI pages changed.
- `1`: a page scored below the threshold.
- `2`: the run hit an error, such as a missing key or an unreachable preview.
- `3`: a page redirected to a login page and needs a saved session.

## Support

[thedesignagent.ai/support](https://www.thedesignagent.ai/support) · [Privacy](https://www.thedesignagent.ai/privacy) · [Terms](https://www.thedesignagent.ai/terms)

## License

MIT
