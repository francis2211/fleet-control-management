# Challenge: Organize the Fleet and Read Fleet Health

## Context

Challenge 1 got the cluster connected — Agent Control installed, a fleet created, the cluster assigned to it. This challenge does not create anything new or push any configuration; it teaches the learner to *read* Fleet Control: the three-part anatomy of a fleet page, where health/deployment/configuration signals live, who is allowed to manage a fleet, and which platforms (Kubernetes vs. Linux/Windows) are actually GA versus public preview. It is deliberately the lightest challenge in the track — pure navigation and observation — so that Challenge 3's first hands-on remote-configuration change lands on a learner who already knows where to look for the result.

## Prior State

- **From Challenge 1 (01-install-agent-control):** Agent Control is installed and running in the `newrelic-agent-control` namespace on the `workstation` k3s cluster (Helm releases `agent-control-bootstrap`, `agent-control-deployment`, `agent-control-cd`, all `deployed`). The cluster is registered as a **managed entity** and — because the fleet ID was supplied at install time — is already an **assigned managed entity** in a fleet named `fleet-control-lab-<participant-id>` (Kubernetes Cluster type), which the learner created themselves in the Fleet Control browser tab. `~/agent-control-values.yaml` on the workstation still contains that fleet's Entity guid under `agentControlDeployment.chartValues.config.fleet_control.fleet_id`, and `/root/fleet-lab-name.txt` holds the session's unique lab name (used as both the fleet name and the `global.cluster` value).
- **Terminology already taught (Challenge 1) — referenced confidently, not re-taught here:** Agent Control as supervisor; system identity; **managed entity**, **fleet**, **assigned** vs. **unassigned** managed entity; Flux as the declarative engine. Note: Challenge 1's assignment text never named a specific New Relic *permission role* to the learner (system identity credentials were simply pre-provided as environment variables) — so this challenge introduces the Organization Manager role as genuinely new, not as a contrast to a role the learner has already seen named.
- **Files present:** `~/agent-control-values.yaml`, `/root/fleet-lab-name.txt` (both from Challenge 1, read but not modified in this challenge).
- **Services running:** unchanged from Challenge 1 — no new services, no config changes in this challenge.

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|----------------|-------------|
| Agent Control is installed and its Helm releases are deployed | workstation | `helm list -n newrelic-agent-control -o json \| jq -e 'map(select(.status=="deployed")) \| length >= 3'` | 01-install-agent-control |
| The cluster is an assigned managed entity in a fleet, and that fleet's ID is recorded locally | workstation | `grep -qv 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml` | 01-install-agent-control |
| The session-unique lab name (used as both fleet name and cluster name) is available | workstation | `test -n "$(cat /root/fleet-lab-name.txt 2>/dev/null)"` | 01-install-agent-control |

## Assignment Outline

Health confirmation is deliberately the *last* step, not the first — it needs the 5-10 minute post-install window from Challenge 1 to have elapsed, and by the time a learner has worked through the rest of this challenge's steps, more of that window has passed. Front-loading the health check risks a false "still not healthy" moment right when the learner has had the least time to wait.

1. **Do (browser, Fleet Control): open the fleet and confirm the entity is there.** Go to **New Relic Control → Fleets**, open `fleet-control-lab-<participant-id>` (the same fleet from Challenge 1 — there's nothing to create here), and land on its **Summary** tab. New concept: every fleet's management page has three components — **Active Deployment**, **Agents & Configurations**, and **Managed Entities** — find your cluster listed in the Managed Entities table. Its instrumentation status may still be catching up; that's fine, this step is just confirming the entity itself is present. Status gets a proper look in step 5.
2. **Read (one line) + do (browser): the Agents tab and effective configuration.** New concept: the **Agents** tab lists every agent running on entities in the fleet — and Agent Control itself is listed here as an agent type, not just the things it manages. Click into the Agent Control entry for your cluster to open its **effective configuration** view. Note the Helm chart version shown.
3. **Do (terminal): cross-check the effective configuration against reality.** Run `helm list -n newrelic-agent-control -o json | jq -r '.[] | select(.name=="agent-control-deployment") | .chart_version // .app_version'` and confirm the version matches what Fleet Control's effective-configuration view just showed. This is the point of that view: what Fleet Control displays is exactly what's running on the cluster, not a cached or intended value.
4. **Read (one line) + do (browser): who's allowed to manage this fleet.** New concept: creating or editing a fleet requires the **Organization Manager** role in your New Relic account. This is the first time a named permission role has come up in the track — Challenge 1's system identity credentials were simply handed to you as environment variables, with no role for you to hold. Find the fleet's **Edit permissions** action (from the fleet list's `...` menu) to see where this shows up in the UI; you don't need to change anything.
5. **Read (one line) + do (browser): platform scope, then confirm health.** New concept: Kubernetes cluster management is **generally available**, while Linux and Windows host management are **public preview** — look for this distinction in the feature-availability banner on the Fleets page; it's why this track's hands-on work stays on Kubernetes throughout. Now return to your fleet's **Summary** tab: the Managed Entities table should show your cluster's instrumentation status as healthy. If it doesn't yet, give it a couple more minutes and refresh — this is the same 5-10 minute connection window Challenge 1 called out.
6. **Do (browser): open the Deployments tab.** Navigate to the fleet's **Deployments** tab and observe that it's empty — no deployment has been created yet. This is expected: Agent Control is running, but no agent (Infrastructure, OpenTelemetry, etc.) has been deployed to it yet. That's Challenge 3.

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | Reused from Challenge 1, unchanged. Used only for the effective-configuration cross-check in step 3. |
| Fleet Control | browser | virtualbrowsers: fleet-control | Reused from Challenge 1, unchanged. This challenge's primary surface — every new concept is read directly from this UI. |

## Scripts

### Setup (`setup-workstation`)

- No new setup is needed. This challenge reads state Challenge 1 already created; nothing new is provisioned, no files are staged, no services are started.
- Verification tail (re-confirms the "Provided by: 01-install-agent-control" prerequisites are still functional at the start of this challenge, in case of hot-start or drift — this challenge doesn't own any of these capabilities, so it asserts rather than establishes them):
  ```bash
  helm list -n newrelic-agent-control -o json | jq -e 'map(select(.status=="deployed")) | length >= 3' >/dev/null \
    || { echo "FATAL: expected Agent Control Helm releases from Challenge 1 are not deployed"; exit 1; }
  grep -qv 'REPLACE_WITH_YOUR_FLEET_ID' ~/agent-control-values.yaml \
    || { echo "FATAL: fleet_id from Challenge 1 is missing or still a placeholder"; exit 1; }
  test -n "$(cat /root/fleet-lab-name.txt 2>/dev/null)" \
    || { echo "FATAL: session lab name from Challenge 1 is missing"; exit 1; }
  ```

### Check (`check-workstation`)

1. **Challenge 1's installed state is still intact (covers steps 1-4 implicitly; also guards Challenge 3).** `helm list -n newrelic-agent-control -o json | jq -e 'map(select(.status=="deployed")) | length >= 3'`. Fail message: "Agent Control's Helm releases from Challenge 1 are no longer all deployed — did something get uninstalled while exploring? Re-run Challenge 1's `helm upgrade --install` command."
2. **Effective-configuration cross-check was actually run (step 3).** The learner ran the `helm list -n newrelic-agent-control | jq ...` command from step 3 in this session (implementer's choice of mechanism — e.g. shell history, or a wrapper script the assignment has them source that timestamps invocation). Fail message: "Run the `helm list -n newrelic-agent-control` command from step 3 to cross-check the chart version Fleet Control showed you."
3. **Agent Control is stably connected (step 5) — verified via local proxy, not the New Relic API.** This track provisions no New Relic User API key, so this check cannot directly query Fleet Control's backend to confirm the UI's "healthy" status the way a live NerdGraph query would; instead it asserts the local signals that should be true if that connection succeeded: (a) every pod in `newrelic-agent-control` is `Running` (`kubectl get pods -n newrelic-agent-control --no-headers | awk '{print $2}' | grep -qv Running` finds nothing not-Running, restricted to pods whose `.status.phase` isn't `Succeeded`/`Completed` for the one-shot install job), (b) none of those pods have restarted more than twice (`kubectl get pods -n newrelic-agent-control -o json | jq -e '[.items[].status.containerStatuses[]?.restartCount] | max <= 2'`), (c) no authentication-failure signature appears in any pod's logs (same check as Challenge 1's check #4: looping over pod names, `grep -qi 'Expired client secret'` finds nothing), and (d) at least 5 minutes have elapsed since the oldest Agent Control pod started (`kubectl get pods -n newrelic-agent-control -o json | jq -r '[.items[].status.startTime] | min'` compared against the current time), covering the same 5-10 minute connection window Challenge 1 called out. This is a meaningfully weaker guarantee than a real backend query — a learner should still visually confirm "healthy" in the UI per step 5's text — but it's the strongest signal available without provisioning API access. Fail message: "Agent Control doesn't look stably connected yet (pods not all Running, a pod has restarted repeatedly, an auth error appeared in the logs, or not enough time has passed) — check `kubectl get pods -n newrelic-agent-control` and the Managed Entities table on your fleet's Summary tab."
4. **No deployment has been pushed yet (step 6), verified via the local proxy for "nothing beyond Agent Control is installed."** `helm list -A -o json | jq -e 'length == 3'` — exactly the three Challenge-1 releases (`agent-control-bootstrap`, `agent-control-deployment`, `agent-control-cd`) and nothing more, confirming no agent has been deployed via a Fleet Control deployment yet. Fail message: "Something beyond Agent Control appears to be installed already — this challenge is read-only; Challenge 3 is where you'll deploy your first agent."
5. *(Coverage note, per script-design)* Steps 1 and 4 (opening the Summary/Agents tabs, reading the permissions callout) are pure informational reads with no infrastructure side effect of their own to assert on directly — checks #1-4 above instead validate the state those reads are describing (entity present and unbroken, effective-config claim cross-checked, health, deployment count), which is the closest a check script can get to confirming the learner engaged with real Fleet Control state rather than just clicking through.

### Solve (`solve-workstation`)

- Idempotent; repairs any of the four check conditions independently rather than assuming they hold:
  1. **Repair check #1 if broken:** if fewer than 3 Helm releases show `deployed` in `newrelic-agent-control`, re-run Challenge 1's exact install command (`helm upgrade --install agent-control-bootstrap newrelic/agent-control-bootstrap -n newrelic-agent-control --create-namespace -f ~/agent-control-values.yaml`) to restore state, rather than assuming a learner working through a read-only challenge couldn't have broken it.
  2. **Satisfy check #2:** run the same `helm list -n newrelic-agent-control -o json | jq ...` command from assignment step 3.
  3. **Satisfy check #3:** if the local proxy signals (all pods Running, restart count low, no auth-failure log line, 5+ minutes elapsed) don't yet hold, poll/wait for them (bounded, e.g. up to 5 additional minutes) rather than fabricating a pass — this challenge's whole point is real Agent Control state, and solve should reflect that same reality, not fake it. Solve cannot independently verify the real Fleet Control UI status (no API key), so it targets exactly the same local signals check #3 tests.
  4. **Check #4 requires no action** — it holds automatically as long as check #1's state is intact and the learner hasn't manually deployed anything (out of scope for this read-only challenge).

### Cleanup (`cleanup-workstation`)

- None. Nothing was created in this challenge; Challenge 1's installed state persists forward into Challenge 3 unchanged.

## Infrastructure Changes

- **None.** No new hosts, ports, browsers, or secrets. This challenge reuses only the `workstation` VM and the `fleet-control` virtual browser from Challenge 1. This track does not provision a New Relic User API key (an earlier draft of Challenge 1's plan proposed one; that decision was reversed — see Challenge 1's Infrastructure Changes), so this challenge's health check (step 5) relies entirely on local `kubectl`/log proxies rather than a live NerdGraph query, as detailed in the Check section above.

## Concepts

- **New (full scaffolding):** the three-part fleet anatomy (Active Deployment / Agents & Configurations / Managed Entities); the Agents tab and "effective configuration" view (and why it matters — it reflects real running state, not intended state); the Organization Manager permission role for managing fleets; GA (Kubernetes) vs. public preview (Linux/Windows) scope.
- **Confident reference (taught in Challenge 1, not re-explained):** Agent Control as supervisor; managed entity; fleet; assigned vs. unassigned managed entity; Flux.
- **Deferred to later challenges:** remote configuration and agent types (Challenge 3); multi-agent fleets (Challenge 4); deployment failure diagnosis (Challenge 5) — this challenge observes an *empty* Deployments tab specifically to set up Challenge 3's first deployment.

## Timing

8 minutes: six lightweight steps — three UI reads-with-navigation (Summary tab, Agents/effective-configuration, permissions/GA-preview-then-health), one terminal cross-check, and a closing Deployments-tab observation. No install wait, no config edits, no Helm commands beyond a single read-only `helm list`. Placing the health confirmation last (step 5, immediately before the equally-quick step 6) rather than first means the challenge's own step 1-4 work — a few minutes of real navigation — does double duty as the buffer for Challenge 1's 5-10 minute connection window, instead of requiring a dedicated wait inside this challenge's estimate. Matches the track roadmap's designation of this challenge as "the shortest challenge in the track."
