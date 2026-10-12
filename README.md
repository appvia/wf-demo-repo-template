# Infrastructure stack template (dev on merge)

A Wayfinder repository template. A repository created from it holds a
Wayfinder stack of cloud resources and nothing else: no application code, no
container image, no cluster. Every merge to `main` runs `wf deploy` against the
`dev` environment.

| When | What happens in the new repository |
| --- | --- |
| A push to `main` (a merge) | `make lint`, then `wf deploy` of `<stack>-dev` into the `dev` environment |

Creating the repository also produces one Wayfinder service account, trusted
only by that repository's `dev` GitHub environment and able to deploy only to
the `dev` Wayfinder environment. It also creates the `dev` GitHub environment
and the variables the workflow reads. Nobody makes a credential or a role
binding by hand. [`skeleton/README.md`](./skeleton/README.md) lists every variable.

## Layout

```
wf-demo-repo-template/
├── RepoTemplate-wf-demo-infra.yaml   you give this to Wayfinder: where the template is
├── wayfinder-template.yaml           Wayfinder reads this: inputs, service account, variables
└── skeleton/                         what gets copied into the new repository
    ├── Wayfinder.yaml                the stack (one example DynamoDB table)
    ├── plans/aws-dynamodb-table.yaml the example CloudResourcePlan
    ├── Makefile                      make lint
    ├── .wayfinder/ci.env             stack name and wf CLI version (rendered)
    ├── .wayfinder/install-wf.sh      installs wf on the runner (raw)
    └── .github/workflows/
        └── wf-deploy-dev.yaml        lint, then wf deploy to dev on push to main (raw)
```

The template is at the root of this repository, so the `RepoTemplate` leaves
`spec.path` empty.

## Inputs

| Input | Default | What it sets |
| --- | --- | --- |
| `description` | empty | The README and the stack's description |
| `region` | `eu-west-2` | The AWS region, as the `WF_REGION` repository variable |

The environment is fixed to `dev` rather than offered as an input. The
workspace the repository is created in must have a `dev` environment. If it
does not, creating the repository succeeds and its service account fails to
apply.

## Use it

```bash
# 1. Push this repository to github.com/appvia/wf-demo-repo-template, which is
#    where RepoTemplate-wf-demo-infra.yaml points.

# 2. Register it, and check Wayfinder could read the manifest
wf apply -f RepoTemplate-wf-demo-infra.yaml
wf get repotemplate wf-demo-infra -o yaml        # status.manifest is populated

# 3. Preview what it renders, creating nothing
wf create stack my-infra --from-template wf-demo-infra --dry-run

# 4. Create the repository and the stack. --github-org picks the connected
#    GitHub organisation to create it under (`wf get githuborgs` lists them),
#    and can be left out when only one is connected.
wf create stack my-infra --from-template wf-demo-infra \
  --input region=eu-west-2 \
  --github-org <githuborg> --workspace <workspace>
```

## Before the first deploy

- A `dev` environment in the workspace.
- A cloud identity `<workspace>/dev/aws-dev`, or set the `WF_IDENTITY`
  variable on the new repository to the one to use. It must be able to create
  what the stack's plans declare: DynamoDB tables, until you replace the
  example.
- The service account manifests applied. Creating the repository hands them to
  you. `wf get coderepo <name> -w <workspace> --configuration` returns them
  again.
