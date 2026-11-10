# Challenge: Install Agent Control and Connect to Fleet Control

## Context

This is the first challenge in the track. The learner arrives with a fresh k3s cluster and no New Relic instrumentation of any kind on it. By the end of this challenge, that cluster is a **managed entity** running **Agent Control** — connected to **Fleet Control** and assigned to a fleet the learner created themselves. Every later challenge (organizing the fleet, remote-configuring the Infrastructure agent, adding OpenTelemetry, troubleshooting a rollout) depends on this connection existing and being healthy, so this challenge is scoped tightly to "get connected," not "explore what's possible once connected" — that's Challenge 2's job.

The design deliberately follows New Relic's own documented "fastest path" for Fleet Control (create the fleet first, then install Agent Control referencing it — see [Set up Fleet Control](https://docs.newrelic.com/docs/new-relic-control/fleet-control/setup)), but skips the browser-driven "Guided Install" wizard's live system-identity generation step. Track-level setup already injects a shared license key and long-lived system identity client credentials as secrets (`NEW_RELIC_LICENSE_KEY`, `NR_SYSTEM_IDENTITY_ORG_ID`, `NR_SYSTEM_IDENTITY_CLIENT_ID`, `NR_SYSTEM_IDENTITY_CLIENT_SECRET`) — the same fields the Guided Install would generate for you, pre-populated so many concurrent learner sessions don't each mint their own short-lived (12-hour) identity against the shared account. The learner still performs the one step that *is* inherently per-session: creating a uniquely-named fleet and wiring its ID into the Helm values themselves.

## Prior State

- No prior challenges — this is Challenge 1.
- Services running: k3s is up and `kubectl` is configured against it (track-level setup). Helm 3 is installed (track-level setup). No New Relic components of any kind are present on the cluster.
- Files present: none related to this track yet.
- State from previous challenges: none.

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|----------------|-------------|
| k3s cluster is ready and `kubectl` is configured against it | workstation | `kubectl get nodes \| grep -q ' Ready'` | track setup |
| Helm 3 is installed | workstation | `helm version --short` | track setup |
| New Relic license key and system identity credentials are available as environment variables | workstation | `test -n "$NEW_RELIC_LICENSE_KEY" && test -n "$NR_SYSTEM_IDENTITY_ORG_ID" && test -n "$NR_SYSTEM_IDENTITY_CLIENT_ID" && test -n "$NR_SYSTEM_IDENTITY_CLIENT_SECRET"` | track setup |
| Workstation can reach New Relic's Fleet Control / OpAMP endpoint | workstation | `curl -sS -o /dev/null -w '%{http_code}' https://opamp.service.newrelic.com/v1/opamp \| grep -qE '^[234]'` | track setup |
| A session-unique lab name is available for naming the fleet and the cluster identity | workstation | `test -n "$(cat /root/fleet-lab-name.txt 2>/dev/null)"` | this challenge |
| A `values.yaml` template pre-filled with license key and system identity credentials exists, awaiting a fleet ID | workstation | `grep -q 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml && grep -q "$(cat /root/fleet-lab-name.txt)" ~/agent-control-values.yaml` | this challenge |
| New Relic Helm chart repo is reachable and indexed locally | workstation | `helm repo list \| grep -q newrelic` | this challenge |

## Assignment Outline

Each concept below is introduced immediately before the step that uses it, not batched up front.

1. **Read (one short paragraph): what Agent Control is, and the one rule that shapes today's order.** Using the docs' own framing: Agent Control is "a lightweight agent supervisor that runs on Kubernetes clusters... to manage the lifecycle of New Relic and open-source instrumentation agents," communicating with Fleet Control, the centralized control plane. State the one rule that determines what happens next: *you must create a fleet before installing Agent Control*, because installation is what assigns the entity to the fleet. That's the only concept needed to motivate step 2 — the rest of the vocabulary arrives where it's used.
2. **Do (browser tab, Fleet Control): create the fleet.** Run `cat /root/fleet-lab-name.txt` in the terminal to get this session's unique lab name (`fleet-control-lab-<participant-id>`). In the Fleet Control browser tab, go to **New Relic Control → Fleets → Create a fleet**, name it exactly that value, set **Type of Fleet** to **Kubernetes Cluster**, and click **Create Fleet**. *Right here*, define the two terms this action just created an instance of: a **fleet** is a homogeneous group of managed entities sharing one instrumentation config, and once the cluster joins it below it becomes an **assigned managed entity** (versus **unassigned**, which is any entity running Agent Control but not yet in a fleet). Open the new fleet and click **Metadata** to reveal its **Entity guid** — the fleet ID needed next.
3. **Read (one line) + do (terminal): system identities.** New Relic Control authenticates services using non-human "system identities" (an `organizationId` + `clientId`/`clientSecret` pair) — already provided as environment variables on the workstation, so there's nothing to generate. Open `~/agent-control-values.yaml` (already staged with the license key, system identity, and cluster name filled in) and replace the `REPLACE_WITH_YOUR_FLEET_ID` placeholder under `agentControlDeployment.chartValues.config.fleet_control.fleet_id` with the Entity guid copied in step 2.
4. **Do (terminal): install via Helm, and see Flux for the first time.** The New Relic Helm repo is already added (setup did this — confirm with `helm repo list`). Install the `agent-control-bootstrap` chart into a dedicated `newrelic-agent-control` namespace using the completed values file:
   ```
   helm upgrade --install agent-control-bootstrap newrelic/agent-control-bootstrap \
     -n newrelic-agent-control --create-namespace \
     -f ~/agent-control-values.yaml
   ```
   This one command triggers a short-lived install Job that installs two more releases into the same namespace — `agent-control-deployment` (the Agent Control supervisor itself) and `agent-control-cd` (an embedded **Flux** instance, the declarative "GitOps-style" engine Agent Control uses on Kubernetes to reconcile agents onto the cluster). The learner never touches Flux directly, but should recognize the name in the next step's output.
5. **Do (terminal): verify locally.** Confirm the Helm releases report `deployed` and the pods in `newrelic-agent-control` come up `Running`:
   ```
   helm list -n newrelic-agent-control
   kubectl get pods -n newrelic-agent-control
   ```
   Call out explicitly that the *cluster* showing "healthy" inside Fleet Control can take 5-10 minutes after the pods are running (New Relic's own docs cite this figure for the connection test) — this is expected, not a stall.
6. **Do (browser, Fleet Control): confirm the connection.** Go back to **Fleets**, open the fleet created in step 2, and check the **Managed Entities** table for the cluster. Note (read-only, no action) that its instrumentation status will read healthy once the connection completes — full exploration of this fleet view is Challenge 2.

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | Default workdir `/root` (or learner home); `~/agent-control-values.yaml` and `/root/fleet-lab-name.txt` staged here by setup. |
| Fleet Control | browser | virtualbrowsers: fleet-control | Points at New Relic login / Fleet Control (`https://one.newrelic.com`, or account-specific URL from track-level config). Reused unchanged from track-level sandbox config; no new browser is added in this challenge. |

## Scripts

### Setup (`setup-workstation`)

- Compute the session-unique lab name from `$INSTRUQT_PARTICIPANT_ID` (fallback to `$_SANDBOX_ID` if the participant ID variable is unavailable in this Instruqt runtime) as `fleet-control-lab-<id>`, sanitized to a DNS-safe, lowercase string (fleet names and the Kubernetes `cluster` field should tolerate the same string). Write it to `/root/fleet-lab-name.txt`.
- Add the New Relic Helm repo and update it: `helm repo add newrelic https://helm-charts.newrelic.com && helm repo update`. This is infrastructure plumbing (identical for every learner), not part of the learning objective, so it is pre-staged rather than left as a learner step. The assignment text reflects this explicitly (step 4 tells the learner the repo is already added and has them confirm it with `helm repo list` rather than re-running `helm repo add`), so the plan doesn't ask the learner to redo what setup already did.
- Render `~/agent-control-values.yaml` from a template, substituting:
  - `global.cluster` → the lab name from `/root/fleet-lab-name.txt`
  - `global.licenseKey` → `$NEW_RELIC_LICENSE_KEY`
  - `agentControlDeployment.chartValues.systemIdentity.organizationId` → `$NR_SYSTEM_IDENTITY_ORG_ID`
  - `agentControlDeployment.chartValues.systemIdentity.parentIdentity.clientId` → `$NR_SYSTEM_IDENTITY_CLIENT_ID`
  - `agentControlDeployment.chartValues.systemIdentity.parentIdentity.clientSecret` → `$NR_SYSTEM_IDENTITY_CLIENT_SECRET`
  - `agentControlDeployment.chartValues.config.fleet_control.fleet_id` → left as the literal placeholder string `REPLACE_WITH_YOUR_FLEET_ID` (this is exactly what the learner must edit — do not pre-fill it, that is the learning objective).
- Do **not** run `helm install`/`helm upgrade` for the bootstrap chart itself, and do not create the fleet in Fleet Control — both are the learner's hands-on steps.
- Verification tail (asserts the two "Provided by: this challenge" prerequisite rows):
  ```bash
  test -n "$(cat /root/fleet-lab-name.txt)" || { echo "FATAL: fleet-lab-name.txt not populated"; exit 1; }
  grep -q 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml || { echo "FATAL: values.yaml missing fleet_id placeholder"; exit 1; }
  grep -q "$(cat /root/fleet-lab-name.txt)" ~/agent-control-values.yaml || { echo "FATAL: values.yaml cluster name not substituted"; exit 1; }
  helm repo list | grep -q newrelic || { echo "FATAL: newrelic helm repo not added"; exit 1; }
  ```

### Check (`check-workstation`)

1. **Fleet ID was filled in.** `grep -qv 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml` (i.e., the placeholder string is gone) and the replacement looks like a GUID (`grep -qE '[0-9a-fA-F-]{20,}' ~/agent-control-values.yaml` scoped to the `fleet_id:` line). Fail message: "Edit `~/agent-control-values.yaml` and replace `REPLACE_WITH_YOUR_FLEET_ID` with the Entity guid of the fleet you created in Fleet Control."
2. **Helm releases deployed.** `helm status agent-control-bootstrap -n newrelic-agent-control` reports `STATUS: deployed`, and the same for the two nested releases it creates, `agent-control-deployment` and `agent-control-cd`, in the same namespace. Fail message: "The `agent-control-bootstrap` Helm release isn't deployed yet in the `newrelic-agent-control` namespace — run the `helm upgrade --install` command from the assignment."
3. **Pods running.** Every pod in `kubectl get pods -n newrelic-agent-control` is `Running` (excluding completed install/uninstall Job pods, which are expected to show `Completed`). Fail message: "Pods in `newrelic-agent-control` aren't all Running yet — check `kubectl get pods -n newrelic-agent-control` and `kubectl describe pod` on anything not Running."
4. **No authentication failure.** Agent Control's own pod logs do not contain the documented failure signature for a bad/expired system identity: `for p in $(kubectl get pods -n newrelic-agent-control -o name); do kubectl logs -n newrelic-agent-control "$p" 2>/dev/null; done | grep -qi 'Expired client secret'` returns nothing (looping over every pod name in the namespace sidesteps needing to know the exact chart-assigned label selector at plan time; the implementer should confirm the precise pod label — e.g. `app.kubernetes.io/name`— at generation time from the actual deployed chart and may tighten this to a label selector then). Fail message: "Agent Control logged an authentication error against Fleet Control — double check the fleet ID and that the system identity environment variables are present (`env | grep NR_SYSTEM_IDENTITY`)."
5. *(Edge case coverage per script-design rubric)* If the release exists but check #1 was skipped (learner ran `helm install` without editing the fleet ID first), check #2 may pass while #1 fails — the check script evaluates all four independently and reports every failing message, not just the first, so a learner who installed with the placeholder still intact gets pointed at exactly what to fix rather than a generic failure.

### Solve (`solve-workstation`)

- Idempotent; safe to run whether the learner has done nothing, has created the fleet but not edited the values file, or has already installed:
  1. If `~/agent-control-values.yaml` still contains `REPLACE_WITH_YOUR_FLEET_ID`, generate a synthetic, GUID-shaped identifier locally (`uuidgen`, or `python3 -c "import uuid; print(uuid.uuid4())"` as a fallback) and substitute it into the values file in place of the placeholder. **This is not a real Fleet Control fleet** — solve has no New Relic API credential to create one, so this satisfies check-workstation's local, format-level assertion (a GUID-shaped string is present) without a live backend object behind it. This is an accepted limitation of the skip path for this challenge: a learner who skips gets a locally-passing environment, not a real Fleet Control-side connection. (Track-level decision: no New Relic User API key secret is provisioned for this track: real fleet/entity verification from check or solve scripts is out of scope everywhere in this track, not just here.)
  2. Run the same `helm upgrade --install agent-control-bootstrap newrelic/agent-control-bootstrap -n newrelic-agent-control --create-namespace -f ~/agent-control-values.yaml` command from Assignment Outline step 4 if the `agent-control-bootstrap` release is not already `deployed` (the `newrelic` repo is already present from setup — no `helm repo add` needed here either).
  3. Poll (`kubectl wait --for=condition=Ready pod --all -n newrelic-agent-control --timeout=300s` or an equivalent loop) until all pods are Running before returning, so a learner who clicks "skip" lands in the same state check-workstation expects.

### Cleanup (`cleanup-workstation`)

- Not required within the track (Challenges 2-5 build directly on this installed state — there is nothing to tear down between challenges).
- Track-level teardown (VM destruction at session end) removes everything; no explicit `cleanup-workstation` script is needed for this challenge.

## Infrastructure Changes

- **No new hosts, ports, or virtual browsers.** This challenge reuses the single `workstation` VM and the single Fleet Control virtual browser tab defined at track level — both already exist in `config.yml` before this challenge's content is added.
- **No new secrets.** An earlier draft of this plan proposed a `NEW_RELIC_USER_API_KEY` secret so `solve-workstation` could create a fleet via NerdGraph when a learner clicks "skip." That's been dropped per an explicit decision not to provision an API key for this track — this challenge (and the track as a whole) uses only the four secrets the track plan already named (`NEW_RELIC_LICENSE_KEY`, `NR_SYSTEM_IDENTITY_ORG_ID`, `NR_SYSTEM_IDENTITY_CLIENT_ID`, `NR_SYSTEM_IDENTITY_CLIENT_SECRET`). Solve now uses a locally-generated synthetic identifier instead of a live API call (see **Solve** above).
- **`config.yml` additions for this challenge:** none — the `virtualmachines:`, `virtualbrowsers:`, and `secrets:` blocks are unchanged from the track-level scaffold this challenge plan assumes already exists.
- **Environment variables:** no new VM-level `environment:` keys are needed; `$INSTRUQT_PARTICIPANT_ID` (or `$_SANDBOX_ID` fallback) is assumed available in the Instruqt VM runtime already, and the four New Relic credential secrets are injected automatically by the `secrets:` block per `config.yml` semantics (available as environment variables in every container/VM).

## Concepts

- **New (full scaffolding):** Agent Control as a lightweight supervisor; system identities (organizationId/clientId/clientSecret) as the auth mechanism between Agent Control and Fleet Control; the managed-entity / fleet / assigned vs. unassigned vocabulary; the install-order constraint (fleet before Agent Control); Flux as the declarative, GitOps-style engine Agent Control uses under the hood on Kubernetes (named/labeled but not operated directly).
- **Confident reference (already covered by track prerequisites, not re-taught):** `kubectl` basics, Helm 3 basics, reading a `values.yaml` file — the track's stated audience prerequisites assume comfort with `kubectl`/`helm`/YAML already.
- **Deferred to later challenges:** fleet health/deployment/entity views in depth (Challenge 2), agent types and remote configuration (Challenge 3), multi-agent fleets (Challenge 4), rollout troubleshooting (Challenge 5).

## Timing

15 minutes: ~5 minutes of hands-on reading/YAML-editing/Helm-command work (steps 1-5), plus the ~5-10 minute unavoidable wait New Relic's own docs describe between Agent Control's pods coming up and the cluster showing healthy in Fleet Control (step 6). The assignment text calls out this wait explicitly so learners don't mistake it for a stall.

**Caveat for the track as a whole (not fixable within this single challenge plan):** the track plan's Challenge Roadmap sums all five challenges' estimates (15+8+10+12+15) to exactly 60 minutes, matching the track's stated `~60 min` duration with no stated buffer for slower learners, setup/teardown, or inter-challenge transitions. This challenge's own 15-minute estimate is realistic and independently defensible (it's driven by a real, externally-documented 5-10 minute connection-test wait, not padding), so shrinking it would misrepresent the work rather than fix the track-level buffer problem. Flagging for the track planner/user: consider configuring the track's actual `timelimit` in `config.yml`/track settings above the literal 60-minute sum (e.g., 75 min) rather than adjusting individual challenge estimates.
