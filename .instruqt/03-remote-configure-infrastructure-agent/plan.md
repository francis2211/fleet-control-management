# Challenge: Remotely Configure the Infrastructure Agent

## Context

Challenges 1 and 2 got the cluster connected and taught the learner to read Fleet Control's views — but nothing has actually been deployed to the cluster yet (Challenge 2 deliberately ended by observing an empty Deployments tab). This challenge is the track's first hands-on configuration change: the learner authors a remote configuration for the New Relic Infrastructure agent (turning on process metrics), pushes it as a deployment, and verifies from the terminal that Agent Control's Flux engine actually reconciled it onto the cluster — all without ever running `helm install` by hand. This is the payoff moment for everything Challenges 1-2 set up: "manage instrumentation lifecycle at scale... without touching individual hosts."

## Prior State

- **From Challenge 1 (01-install-agent-control):** Agent Control is installed and running in `newrelic-agent-control` (Helm releases `agent-control-bootstrap`, `agent-control-deployment`, `agent-control-cd`, all `deployed`). The cluster is an assigned managed entity in fleet `fleet-control-lab-<participant-id>`. `~/agent-control-values.yaml` and `/root/fleet-lab-name.txt` are present. No New Relic User API key is provisioned for this track (see Challenge 1's Infrastructure Changes) — check and solve scripts in this challenge rely on `kubectl`/Flux state only.
- **From Challenge 2 (02-explore-fleet-control):** the learner has confirmed the cluster reports healthy in Fleet Control and has already used the Agents tab's effective-configuration view and the fleet's Summary/Deployments tabs — so the *navigation* to these UI areas is confident reference, not new, in this challenge. The Deployments tab was observed empty at the end of Challenge 2; this challenge is what fills it in.
- **Terminology already taught — referenced confidently, not re-taught:** Agent Control as supervisor; managed entity; fleet; assigned vs. unassigned managed entity; Flux (named in Challenge 1, not yet operated); the three-part fleet anatomy (Active Deployment / Agents & Configurations / Managed Entities) and the effective-configuration view (Challenge 2).
- **Files present, unchanged from Challenge 1:** `~/agent-control-values.yaml`, `/root/fleet-lab-name.txt`.
- **Services running:** unchanged from Challenge 1/2 at challenge start — Agent Control only, no sub-agents deployed yet. This challenge is what changes that.

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|----------------|-------------|
| Agent Control's Helm releases are deployed | workstation | `helm list -n newrelic-agent-control -o json \| jq -e 'map(select(.status=="deployed")) \| length >= 3'` | 01-install-agent-control |
| The cluster is an assigned managed entity in a fleet, and that fleet's ID is recorded locally | workstation | `grep -qv 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml` | 01-install-agent-control |
| The session-unique lab name is available (reused as a naming prefix for this challenge's Configuration, since Configurations are global to the shared account) | workstation | `test -n "$(cat /root/fleet-lab-name.txt 2>/dev/null)"` | 01-install-agent-control |
| No sub-agent has been deployed to the cluster yet (clean starting point for this challenge's own check to measure against) | workstation | `helm list -A -o json \| jq -e 'length == 3'` | 02-explore-fleet-control |

## Assignment Outline

1. **Read (one short paragraph): local vs. remote configuration.** Recall that Agent Control started Challenge 1 with only a *local* configuration (whatever was in `~/agent-control-values.yaml` at install time — no agents at all). Because the cluster is an **assigned** managed entity (Challenge 1), it now takes its configuration from Fleet Control instead — that's precisely what "assigned" means in practice, not just a status label. Today's task is to give it something to receive: a remote configuration for the Infrastructure agent.
2. **Do (browser, Fleet Control): author the configuration.** Go to **New Relic Control → Configurations → Create configuration**. Set **Managed entity type** to **Kubernetes Cluster** and **Agent type** to **New Relic Infrastructure**. Name it `$(cat /root/fleet-lab-name.txt)-infra` — *specifically because Configurations are global objects shared across the whole account* (unlike a fleet, which you scoped to your session already), a generic name would collide with every other concurrent learner's configuration of the same agent type. Click **Continue**, click **Copy** on the right-hand template pane, and paste it into the left-hand authoring pane.
3. **Do (terminal-free, still in the browser editor): make the one change that matters.** In the pasted template, find the `chart_values` → `newrelic-infrastructure` block and set `enableProcessMetrics: true` (this is exactly the `chart_values` structure the concept above referred to — a nested map keyed by the underlying chart name). Click **Save**.
4. **Do (browser): create and start a deployment.** Back on your fleet's page, click **Create a deployment**, give it a name, click **Add an agent**, select **New Relic Infrastructure** — this is the **agent type** (`newrelic/com.newrelic.infrastructure`) the concept above named — pick the latest version, click **Next**, select the configuration you just saved, **Add to deployment**, **Save draft**, then **Start deployment** and confirm.
5. **Do (terminal): watch Flux reconcile it.** Run `kubectl get pods -n newrelic -w` and watch for Infrastructure agent pod(s) to appear and reach `Running` (Ctrl-C once they do). This is Flux — the same engine named back in Challenge 1 — turning the remote configuration you just pushed into an actual `HelmRelease`. Confirm the object exists with `kubectl get helmrelease -A`.
6. **Do (browser): close the loop.** Back in Fleet Control, watch the deployment's status move from **IN PROGRESS** to **COMPLETED**, then open the Infrastructure agent's effective configuration on the Agents tab (same view from Challenge 2) and confirm `enableProcessMetrics: true` is reflected there as *running* configuration, not just what you typed into the editor.

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | Reused from Challenges 1-2, unchanged. Used for step 5's Flux/pod verification. |
| Fleet Control | browser | virtualbrowsers: fleet-control | Reused from Challenges 1-2, unchanged. Primary surface for authoring the configuration and running the deployment. |

## Scripts

### Setup (`setup-workstation`)

- No new setup. This challenge builds entirely on Challenge 1/2 state; nothing is pre-provisioned, and the whole point is for the learner to author the configuration and trigger the deployment themselves.
- Verification tail (re-confirms Challenge 1/2's prerequisites are still functional, and specifically that no agent has already been deployed — this challenge's own check script measures against the *change* from "3 releases" to "4+ releases," so a hot-started or drifted environment that already has a 4th release would break that measurement):
  ```bash
  helm list -n newrelic-agent-control -o json | jq -e 'map(select(.status=="deployed")) | length >= 3' >/dev/null \
    || { echo "FATAL: expected Agent Control Helm releases from Challenge 1 are not deployed"; exit 1; }
  grep -qv 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml \
    || { echo "FATAL: fleet_id from Challenge 1 is missing or still a placeholder"; exit 1; }
  helm list -A -o json | jq -e 'length == 3' >/dev/null \
    || { echo "FATAL: expected exactly 3 Helm releases (Challenge 1 state only) at Challenge 3 start — an agent appears to already be deployed"; exit 1; }
  ```

### Check (`check-workstation`)

1. **A new HelmRelease was reconciled (steps 2-4 collectively — the config was authored and the deployment was pushed), targeting the sub-agents namespace, not Agent Control's own.** Per Agent Control's own agent-type schema (confirmed against `newrelic/newrelic-agent-control`'s `docs/agent_type.md`: Kubernetes sub-agents are deployed as a Flux `HelmRepository`/`HelmRelease` pair with `spec.targetNamespace` set to the sub-agents namespace), assert: `kubectl get helmrelease -A -o json | jq -e '.items[] | select(.spec.targetNamespace=="newrelic" or .metadata.namespace=="newrelic") | select(.status.conditions[]?.type=="Ready" and .status.conditions[]?.status=="True")'` finds at least one match. This deliberately doesn't depend on knowing the exact chart name Agent Control resolves for "New Relic Infrastructure" (candidates are `newrelic-infrastructure`, per the current Terraform-guide example's `chart_values` key, or an umbrella chart like `nri-kubernetes` with that as a subchart — genuinely ambiguous from docs alone) — it distinguishes this release from Agent Control's own bootstrap-related `HelmRelease`s (which target/live in `newrelic-agent-control`, not `newrelic`) purely by namespace, which the schema confirms is a stable signal regardless of which chart name resolves. Fail message: "No Flux `HelmRelease` targeting the `newrelic` namespace was found (or it isn't Ready yet) — confirm you clicked Start deployment in step 4, and that enough time has passed for Flux to reconcile it."
2. **Infrastructure agent pods are running (step 5).** `kubectl get pods -n newrelic --no-headers | awk '{print $3}' | grep -qv Running` returns nothing (i.e., every pod in the `newrelic` sub-agents namespace is Running) and at least one pod exists (`kubectl get pods -n newrelic --no-headers | wc -l` >= 1). Fail message: "No Infrastructure agent pods are Running in the `newrelic` namespace yet — check `kubectl get pods -n newrelic` and `kubectl describe pod` on anything not Running."
3. **Process metrics are actually enabled, not just typed into the editor (step 3's edit took effect).** `kubectl get helmrelease -A -o json | jq -e '[.items[].spec.values] | tostring | test("enableProcessMetrics.*true"; "i")'` returns true — i.e., the deployed `HelmRelease`'s values (not just the configuration the learner typed) actually contain the flag. Fail message: "The deployed configuration doesn't show `enableProcessMetrics: true` — double check you edited the `chart_values.newrelic-infrastructure` block in step 3 before saving, not a different section of the template."
4. **No regression in Challenge 1's base state.** `helm list -n newrelic-agent-control -o json | jq -e 'map(select(.status=="deployed")) | length >= 3'` still passes.

### Solve (`solve-workstation`)

- Idempotent. This track provisions no New Relic User API key, so solve cannot call Fleet Control's API to author a Configuration or start a Deployment the way an earlier draft of this plan proposed. Instead, solve reaches the same *cluster-side* end state directly, bypassing Fleet Control entirely — this works because Flux (Agent Control's embedded CD engine, already running from Challenge 1) reconciles any `HelmRelease`/`HelmRepository` object present in the cluster regardless of whether Agent Control or something else created it:
  1. If check #1 is not yet satisfied, `kubectl apply` the following manifest, confirmed against Agent Control's own agent-type schema documentation (`newrelic/newrelic-agent-control`'s `docs/agent_type.md`, "Kubernetes Deployment" section) rather than guessed — this is exactly the `HelmRepository`/`HelmRelease` pair shape Agent Control itself generates for a Kubernetes sub-agent, applied directly instead of through Fleet Control:
     ```yaml
     apiVersion: source.toolkit.fluxcd.io/v1
     kind: HelmRepository
     metadata:
       name: solve-infrastructure
       namespace: newrelic-agent-control
     spec:
       interval: 30m
       url: https://helm-charts.newrelic.com
     ---
     apiVersion: helm.toolkit.fluxcd.io/v2
     kind: HelmRelease
     metadata:
       name: solve-infrastructure
       namespace: newrelic-agent-control
     spec:
       interval: 3m
       chart:
         spec:
           chart: newrelic-infrastructure
           sourceRef:
             kind: HelmRepository
             name: solve-infrastructure
       targetNamespace: newrelic
       values:
         enableProcessMetrics: true
     ```
     *(Implementer note — narrowed to a single, well-scoped unknown: the `chart:` field's value is the one part of this manifest not fully confirmed from docs alone. The current Terraform-guide example uses the values key `newrelic-infrastructure` for this agent, which is why that's the value used above, but New Relic's own Helm chart index marks the standalone `newrelic-infrastructure` chart deprecated in favor of an `nri-kubernetes`-based chart with `newrelic-infrastructure` as a subchart — if the flat chart name doesn't reconcile with a real deployment, the fix is likely nesting `values` one level deeper under a `newrelic-infrastructure:` key rather than restructuring the whole manifest. Confirm the working value once by completing this challenge manually via the real Fleet Control UI and inspecting the resulting `HelmRelease`, then hard-code it here — check #1 itself doesn't depend on this value, since it matches on `targetNamespace` alone, so getting this one field right only affects whether solve's applied release actually becomes `Ready`.)*
  2. Poll (bounded, e.g. up to 5 minutes) for the resulting `HelmRelease` to go `Ready` and for pods in `newrelic` to reach `Running`, mirroring assignment step 5 rather than faking the state.
  3. If check #4 is broken (Challenge 1's releases were uninstalled), repair by re-running Challenge 1's install command first, before attempting step 1 above.

### Cleanup (`cleanup-workstation`)

- None. The deployed Infrastructure agent and its configuration persist forward into Challenges 4-5, which build on this exact state (Challenge 4 adds a second agent alongside it; Challenge 5 troubleshoots a failed rollout against this same fleet).

## Infrastructure Changes

- **None.** No new hosts, ports, browsers, or secrets. Reuses only the `workstation` VM and `fleet-control` browser from Challenge 1. This track does not provision a New Relic User API key (an earlier draft proposed one for solve automation; that decision was reversed — see Challenge 1's Infrastructure Changes), so this challenge's solve script reaches the cluster's own Flux/HelmRelease state directly rather than calling Fleet Control's API, as detailed in the Solve section above.

## Concepts

- **New (full scaffolding):** local vs. remote configuration and what "assigned" actually changes in practice; agent types and the `newrelic/com.newrelic.infrastructure` identifier; the `chart_version`/`chart_values` structure of a configuration; Flux turning a pushed deployment into a `HelmRelease` (Flux was named in Challenge 1 but not operated until now); Configurations being global to the account, requiring session-unique naming (a direct consequence of the track's shared-account model).
- **Confident reference (Challenges 1-2, not re-explained):** fleet, managed/assigned entity, Agent Control as supervisor, the fleet's three-part anatomy, the effective-configuration view.
- **Deferred to later challenges:** adding a second, different agent type alongside this one (Challenge 4); diagnosing a deployment that fails partway (Challenge 5).

## Timing

**Proposed: 12 minutes — a 2-minute increase over the track plan's originally stated 10-minute figure for this challenge.** Rationale: steps 2-4 alone involve roughly 15 distinct UI actions across two separate, first-time flows — the Configurations "Create configuration" modal and two-pane editor (navigate, open modal, set two dropdowns, type a dynamic session-specific name that requires a terminal round-trip to `cat /root/fleet-lab-name.txt`, Continue, Copy the template, paste, locate and edit the nested `chart_values.newrelic-infrastructure.enableProcessMetrics` field, Save), followed immediately by the "Create a deployment" wizard (name it, Add an agent, pick agent type, pick version, Next, select the saved configuration, Add to deployment, Save draft, Start deployment, confirm). At a realistic pace for an audience new to Fleet Control specifically (per the track's audience section), that's 7-8 minutes on its own, before step 1's reading, step 5's terminal/Flux verification (~2-3 minutes, shorter than Challenge 1's 5-10 minute cloud connection test since this is purely in-cluster Helm reconciliation), and step 6's closing check. 10 minutes was tight enough to risk ~1.3-1.5x actual completion time; 12 minutes keeps this challenge at or below Challenge 4's 12-minute figure (which involves a comparable deployment-wizard flow it can reuse, already-learned, on top of more YAML surface) while still fitting the intermediate 8-15 minute band.

**Flagging for the user/track-planner, not resolving unilaterally:** this 2-minute increase pushes the track's total from the already-tight 60 minutes (flagged as a zero-buffer issue during Challenge 1's planning) to 62 minutes — making that pre-existing caveat marginally worse rather than better. This is a genuine per-challenge correction, not scope creep, but it does compound the track-level timing question already raised. Recommend resolving both at once when the user reviews the track's overall `timelimit` setting: either accept 62 minutes as the new sum and configure the track's timelimit with real buffer above it (e.g. ~75-80 min), or look for time to trim elsewhere in the roadmap (Challenges 4-5) rather than compressing this challenge's realistic estimate back down.
