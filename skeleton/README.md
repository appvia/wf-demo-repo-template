# ${{ .Stack.Name }}

${{ .Inputs.description }}

Infrastructure only: cloud resources declared in a Wayfinder stack, with no
application code, container image or cluster. Every merge to `main` deploys
it to the `dev` environment with `wf deploy`.

| When | What happens |
| --- | --- |
| A push to `main` (a merge) | `make lint`, then `wf deploy` of `${{ .Stack.Name }}-dev` into the `dev` environment |

Nothing else deploys. A pull request runs no Wayfinder job, and there is no
production pipeline.

## Layout

| | |
| --- | --- |
| `Wayfinder.yaml` | The stack: the components to deploy and how they connect |
| `plans/` | The CloudResourcePlans the stack deploys, as `file:plans/...`. Yours to edit |
| `Makefile` | `make lint`: every YAML file parses, and every plan `Wayfinder.yaml` names exists |
| `.wayfinder/ci.env` | The stack name and the `wf` build CI installs |
| `.wayfinder/install-wf.sh` | Installs `wf` on the runner |
| `.github/workflows/wf-deploy-dev.yaml` | Lint, then deploy to `dev` on a push to `main` |

## Adding infrastructure

The stack starts with one example component, `table`, a pay-per-request
DynamoDB table named `${{ .Stack.Name }}-dev-data`. To add your own:

1. Put a CloudResourcePlan in `plans/`, or use one from the catalogue.
2. Add a component to `Wayfinder.yaml` that names it, with
   `plan: file:plans/<plan>.yaml` or `plan: <name>@<version>`.
3. Merge. The next deploy applies the change to `${{ .Stack.Name }}-dev`.

Delete `table` and `plans/aws-dynamodb-table.yaml` when you no longer need the
example. On the next deploy Wayfinder destroys the table.

## How CI signs in

No credential is stored in this repository. The deploy job declares
`environment: dev`; GitHub mints an OIDC token naming that environment, and
`wf` exchanges it for the Wayfinder service account in
`WAYFINDER_SERVICE_ACCOUNT`. That account trusts only this repository's `dev`
environment, and can deploy only to the `dev` Wayfinder environment.

Wayfinder set these when it created this repository:

| Variable | Set on | What it is |
| --- | --- | --- |
| `WAYFINDER_SERVER` | The repository | Wayfinder API URL |
| `WF_REGION` | The repository | AWS region, `${{ .Inputs.region }}` |
| `WAYFINDER_SERVICE_ACCOUNT` | The `dev` environment | The account the deploy job signs in as, `tenant:workspace:name` |
| `WAYFINDER_ENVIRONMENT` | The `dev` environment | `dev` |

One more is yours, and optional:

| Variable | What it does |
| --- | --- |
| `WF_IDENTITY` | Cloud identity to deploy through. Without it, `<workspace>/dev/aws-dev` |

## Before the first deploy

- **The service account must exist.** Creating this repository produced its
  manifests: a ServiceAccount, its federated credential, and the `deployer`
  role in `dev` plus `workspace.cataloguemanagement` in the workspace. If they
  were not applied when the repository was created, get them with
  `wf get coderepo <name> -w <workspace> --configuration` and apply them.
- **A cloud identity for `dev`**, `<workspace>/dev/aws-dev` unless
  `WF_IDENTITY` names another, able to create what `plans/` declares.

## Deploying it yourself

```bash
wf deploy -f Wayfinder.yaml -i ${{ .Stack.Name }}-dev \
  -w <workspace> -e dev \
  --identity <workspace>/dev/aws-dev --region ${{ .Inputs.region }}
```

Add `--dry-run` to plan without applying. Tear it down with
`wf down -i ${{ .Stack.Name }}-dev -w <workspace> -e dev`.
