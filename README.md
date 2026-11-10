# Fleet Control: Centralized Management of New Relic Agents

An [Instruqt](https://instruqt.com) track that teaches New Relic **Fleet Control** hands-on. Learners install **Agent Control** on a Kubernetes cluster, connect it to Fleet Control, and manage the New Relic Infrastructure agent remotely without touching the cluster by hand.

Estimated time: ~60 minutes. The track `timelimit` is 75 minutes to leave a buffer.

## Challenges

| # | Slug | Difficulty | Time | What the learner does |
|---|------|-----------|------|-----------------------|
| 1 | `01-install-agent-control` | basic | 15 min | Create a fleet, add its ID to the Helm values, install Agent Control, confirm the connection |
| 2 | `02-explore-fleet-control` | basic | 8 min | Read fleet health, deployments, effective configuration and permissions (read-only) |
| 3 | `03-remote-configure-infrastructure-agent` | intermediate | 12 min | Author a remote configuration, start a deployment, watch Flux apply it |

The track plan also lists two more challenges, which are not generated yet:

- `04-add-otel-collector`
- `05-roll-out-and-troubleshoot`

The plans are in [.instruqt/](.instruqt/).

## Repository layout

```
track.yml                  Track metadata (slug, title, timelimit, owner)
config.yml                 Sandbox: one k3s VM, a Fleet Control virtual browser, secrets
track_scripts/             Track-level setup (installs Helm and jq, checks prerequisites)
NN-<slug>/                 One directory per challenge
  assignment.md            Learner-facing instructions and tabs
  setup-workstation        Runs when the challenge starts
  check-workstation        Validates the learner's work (uses fail-message)
  solve-workstation        Skip path: brings the environment to the solved state
  cleanup-workstation      Runs when the challenge ends
.instruqt/                 Track and challenge plans (design docs)
.github/                   CI: reusable actions and workflows
```

## Sandbox

- `workstation`: `instruqt/k3s-v1-33-4`, `n1-standard-4`, nested virtualization on.
- `fleet-control`: a virtual browser pointed at `https://one.newrelic.com`.

### Secrets

`config.yml` contains placeholder values for the shared New Relic account's credentials. Replace them before pushing the track:

| Secret | Purpose |
|--------|---------|
| `NEW_RELIC_LICENSE_KEY` | License key passed to Agent Control |
| `NR_SYSTEM_IDENTITY_ORG_ID` | System identity organization ID |
| `NR_SYSTEM_IDENTITY_CLIENT_ID` | System identity client ID |
| `NR_SYSTEM_IDENTITY_CLIENT_SECRET` | System identity client secret |

Do not commit real values. The system identity needs Auth Domain Manager and Organization Manager permissions.

All learner sessions share one New Relic account. Setup derives a session-unique lab name, `fleet-control-lab-<participant-id>`, so fleets and configurations don't collide.

## Known limitations

- No New Relic User API key is provisioned, so the check and solve scripts can't query Fleet Control. They verify cluster state (Helm, pods, Flux `HelmRelease`) instead. In challenge 1, solve inserts a GUID-shaped fleet ID that doesn't correspond to a real fleet.
- Challenge 3's solve script applies a `HelmRelease` directly. Its chart name and values nesting still need confirming against a real run.
- Challenge 1's Helm values keys follow the plan and aren't yet verified against the live `agent-control-bootstrap` chart.

## CI

The workflows in `.github/workflows/` use the `instruqt/cli` Docker image and need an `INSTRUQT_TOKEN` repository secret.

| Workflow | Trigger | Does |
|----------|---------|------|
| `dev-track-test.yml` | Pull request | Convert to `-dev`, validate, push, test |
| `prod-track-test.yml` | Push to `main` | Validate, push, test; commits the track ids on first run |
| `create-dev-track.yml` | Manual | Push the `-dev` variant to Instruqt |
| `promote-dev-track.yml` | Manual (not on `main`) | Pull the dev track and restore the prod identity |

Note that `track test` runs the challenge scripts against the live New Relic account.

## Local development

With the [Instruqt CLI](https://docs.instruqt.com/reference/cli) and `INSTRUQT_TOKEN` set:

```bash
instruqt track validate
instruqt track push
instruqt track test
```

Terminology: **New Relic Control** is the product. **Agent Control** is the supervisor installed on the cluster. **Fleet Control** is the web control plane.
