# Track: Fleet Control: Centralized Management of New Relic Agents

## Terminology note

New Relic's current documentation brands this capability as **"New Relic Control"**, with two component parts:
- **Agent Control** — the lightweight supervisor installed once per Kubernetes cluster or host, which connects to the control plane and manages the lifecycle of instrumentation agents.
- **Fleet Control** — the centralized web UI/control plane (part of New Relic Control) where fleets, remote configuration, and rollouts are managed.

This matches the user's original description ("Fleet Control provides centralized, remote management for your New Relic agents...") almost verbatim — that phrase appears in New Relic's own Fleet Control overview docs. The plan below uses **Fleet Control** as the primary term (matching the user's framing and the docs' UI section name) and introduces **Agent Control** explicitly as the supervisor component learners install first. Worth flagging to the user: some marketing material may use "Fleet Control" more loosely than the docs' precise "Agent Control vs. Fleet Control" split — this plan follows the docs' precise split.

## Intent

Learners install Agent Control on a Kubernetes cluster, connect it to Fleet Control, and use Fleet Control to manage the full instrumentation lifecycle at scale: organizing managed entities into fleets, pushing remote agent configuration, adding new managed agents, and monitoring/troubleshooting rollout health — all without touching individual hosts by hand.

## Products Covered

- New Relic Control / Fleet Control
- Agent Control
- New Relic Infrastructure Agent
- New Relic Distribution of OpenTelemetry (NRDOT / OpenTelemetry Collector)

## Learning Objectives

1. Install and connect Agent Control on a Kubernetes cluster using Helm, license key, and system identity credentials, turning the cluster into a managed entity.
2. Organize managed entities into a fleet and interpret Fleet Control's fleet health, deployment, and entity status views.
3. Deploy and remotely configure a New Relic agent (Infrastructure agent) across a fleet using Fleet Control's remote configuration, without touching the cluster directly.
4. Add a second managed agent (OpenTelemetry Collector / NRDOT) to a fleet's configuration and verify Agent Control reconciles the cluster to match the new desired state.
5. Diagnose and remediate a failed configuration rollout using Fleet Control's health and deployment-error signals.

## Target Audience

- **Role:** Site reliability engineers and platform engineers who already run New Relic agents (Infrastructure agent, APM, OpenTelemetry) in their environment and are responsible for keeping them updated and consistently configured across many hosts/clusters.
- **Experience level:** Intermediate with New Relic (has a New Relic account, has installed at least one agent before, understands MELT/telemetry basics); intermediate with Kubernetes (comfortable with `kubectl`, has used Helm); new to Fleet Control specifically.
- **Prerequisites:**
  - Has a New Relic account and has previously installed a New Relic agent manually (Infrastructure agent or APM) — this track does **not** re-teach what the Infrastructure agent or OpenTelemetry Collector do.
  - Comfortable running `kubectl` and `helm` commands and reading YAML.
  - Does **not** need prior Flux/GitOps or OpAMP protocol knowledge — the track introduces just enough of "how Agent Control reconciles config" to explain what learners observe.
- **Assumed knowledge boundary:** the track assumes New Relic agent fundamentals and basic Kubernetes/Helm literacy; it does not assume familiarity with Flux CD, system identities, or New Relic's newer control-plane concepts — those are taught fresh, starting from Challenge 1.

## Sandbox Requirements

- **Infrastructure pattern:** single-vm-kind-k3s (k3s, pre-baked `instruqt/k3s-v1-33-4` image) + a virtual browser (external URL) pointed at New Relic login / Fleet Control. Kubernetes is chosen deliberately over Linux/Windows hosts: Fleet Control's Kubernetes management is **generally available**, while Linux and Windows host management are in **public preview** (per New Relic's docs, retrieved 2026-09-21) — GA surface area is more stable to build and maintain a lab against, and Kubernetes alone is sufficient to teach the full lifecycle (inventory, fleets, remote config, multi-agent rollout, health monitoring) the user asked for. Windows in particular currently has no MSI/MSIX installer (PowerShell-script-only) and heavier prerequisites, making it a poor fit for a repeatable hands-on lab. This is a deliberate scope decision to confirm with the user — Linux/Windows fleet types can be mentioned conceptually in assignment text without hands-on host provisioning.
- **Hosts:**
  - 1 VM `workstation` — k3s pre-installed and ready, Helm 3 installed by track setup, `n1-standard-4` minimum (Kubernetes base requirement per `single-vm-kind-k3s` guidance), `nested_virtualization: true`.
  - 1 virtual browser (external) — points at New Relic login / Fleet Control (`https://one.newrelic.com` or account-specific URL), used for all Fleet Control UI work (creating fleets, pushing remote config, reading health/deployment status). Paired with the workstation terminal for `kubectl`/`helm` verification — this is the standard "SaaS product walkthrough + terminal" pattern.
- **Cloud accounts:** None (no AWS/Azure/GCP). Instead, an external SaaS dependency: a real New Relic account with Fleet Control access.
  - Required from the account: a **license key**, and **system identity** client credentials (`organizationId`, `clientId`, `clientSecret`) with **Auth Domain Manager** permissions (needed to connect Agent Control to Fleet Control) and **Organization Manager** permissions (needed to create/edit fleets).
  - These must be supplied via config.yml's `secrets:` block at generation time — **do not hardcode real or fake key values in this plan or in any generated script.** Placeholder secret names only (e.g. `NEW_RELIC_LICENSE_KEY`, `NR_SYSTEM_IDENTITY_ORG_ID`, `NR_SYSTEM_IDENTITY_CLIENT_ID`, `NR_SYSTEM_IDENTITY_CLIENT_SECRET`) until the user confirms real values to inject.
  - **Open question for the user to confirm at generation time:** is this one shared New Relic account across all concurrent learner sessions, or a pool of accounts checked out per session (see `account-pool-checkout` pattern)? A shared account is simpler but every learner's fleet/entity names will collide unless each session creates uniquely-named resources. Recommended default absent other guidance: keep a single shared account, but have setup scripts derive a unique fleet name per session (e.g. `fleet-control-lab-${INSTRUQT_PARTICIPANT_ID}` or similar session token) so concurrent learners don't collide in the same New Relic account. This needs explicit user sign-off before generation.
  - Also note: system identity client credentials created via guided install expire after 12 hours per the docs — not a concern for a single ~70-minute session, but worth knowing if hot-start/session reuse is ever considered.
- **Estimated startup time:** ~30-45s for the VM (pre-baked k3s image boots fast). Challenge 1 itself has an additional ~2-5 minute wait for Agent Control's Helm install and Flux reconciliation to complete and for the cluster to show "healthy" in Fleet Control (New Relic's own docs estimate 5-10 minutes for the connection test) — call this out explicitly in Challenge 1's assignment/notes so learners don't think the track has stalled.
- **Special requirements:**
  - `nested_virtualization: true` (required for k3s on an Instruqt VM).
  - Outbound HTTPS egress from the workstation VM to New Relic's control-plane endpoints (`opamp.service.newrelic.com`, `system-identity-oauth.service.newrelic.com`, `download.newrelic.com`, plus regional variants) — confirm the sandbox network policy permits this before generation.

## Track-Level Prerequisites

| Capability (functional after track setup) | Host | Verify command |
|--------------------------------------------|------|----------------|
| k3s cluster is ready and `kubectl` is configured against it | workstation | `kubectl get nodes \| grep -q ' Ready'` |
| Helm 3 is installed | workstation | `helm version --short` |
| New Relic license key and system identity credentials are available as environment variables | workstation | `test -n "$NEW_RELIC_LICENSE_KEY" && test -n "$NR_SYSTEM_IDENTITY_CLIENT_ID" && test -n "$NR_SYSTEM_IDENTITY_CLIENT_SECRET"` |
| Workstation can reach New Relic's Fleet Control / OpAMP endpoint | workstation | `curl -sS -o /dev/null -w '%{http_code}' https://opamp.service.newrelic.com/v1/opamp \| grep -qE '^[234]'` |

## Track Configuration

- **Duration:** ~60 min
- **Difficulty:** Basic → Advanced (progressive)
- **Pausable:** No (default)
- **Show timer:** Yes (default)
- **Hot start:** Not needed — session-scoped fleet/entity naming and time-limited system identity credentials make cross-session reuse impractical; each session should provision fresh.

## Challenge Roadmap

### Challenge 1: Install Agent Control and Connect to Fleet Control

| Field | Value |
|-------|-------|
| **Slug** | 01-install-agent-control |
| **Goal** | Install Agent Control on the k3s cluster via Helm using the provided license key and system identity credentials, then confirm the cluster is connected to New Relic as a managed entity. |
| **Concepts** | Agent Control as a lightweight supervisor; system identities; the Helm-based install flow; managed entity vs. fleet vs. assigned/unassigned entity terminology; Flux as the declarative engine underneath Agent Control on Kubernetes. |
| **Time** | 15 min (longer than its basic difficulty would otherwise suggest — budgeted as ~5 min hands-on YAML/Helm work plus the 5-10 min unavoidable wait New Relic's docs describe for Helm install, Flux reconciliation, and the connection test to go healthy) |
| **Difficulty** | basic |

### Challenge 2: Organize the Fleet and Read Fleet Health

| Field | Value |
|-------|-------|
| **Slug** | 02-explore-fleet-control |
| **Goal** | In Fleet Control, create/select the fleet, confirm the cluster shows as an assigned managed entity with healthy instrumentation status, and inspect the fleet's Active Deployment, Agents & Configurations, and Managed Entities views. |
| **Concepts** | What a fleet is (a homogeneous grouping of managed entities sharing one instrumentation config); fleet permission model (Organization Manager); Fleet Control's real-time visibility (health, deployment progress, configuration errors); GA (Kubernetes) vs. public preview (Linux/Windows) scope. |
| **Time** | 8 min (pure navigation/observation, no install wait, no config changes — the shortest challenge in the track) |
| **Difficulty** | basic |

### Challenge 3: Remotely Configure the Infrastructure Agent

| Field | Value |
|-------|-------|
| **Slug** | 03-remote-configure-infrastructure-agent |
| **Goal** | Add the Infrastructure agent to the fleet's remote configuration in Fleet Control (enabling process metrics), push the change, and verify from the terminal that it synced to the cluster. |
| **Concepts** | Local vs. remote configuration and precedence rules; agent types (`newrelic/com.newrelic.infrastructure`); `chart_version`/`chart_values` structure; how remote config propagates via Flux-managed `HelmRelease` objects. |
| **Time** | 10 min |
| **Difficulty** | intermediate |

### Challenge 4: Add a Second Managed Agent (OpenTelemetry Collector)

| Field | Value |
|-------|-------|
| **Slug** | 04-add-otel-collector |
| **Goal** | Add the New Relic OpenTelemetry Collector (NRDOT) as a second managed agent in the fleet's configuration, and confirm Agent Control/Flux deploys it automatically — with no direct `kubectl`/`helm` install commands. |
| **Concepts** | Multi-agent fleet configuration; the agent type catalog; fleet-wide bulk actions ("upgrade entire fleets in a few clicks, no scripts"); the GitOps-style reconcile loop (desired state from Fleet Control vs. running state on the cluster). |
| **Time** | 12 min (more YAML surface than Challenge 3 — a full new agent block rather than one config flag — plus another Flux reconciliation wait) |
| **Difficulty** | intermediate |

### Challenge 5: Roll Out a Change and Troubleshoot a Failure

| Field | Value |
|-------|-------|
| **Slug** | 05-roll-out-and-troubleshoot |
| **Goal** | Push a remote configuration change that fails to apply cleanly (e.g., an invalid config value), observe Fleet Control surface the resulting configuration/deployment error and unhealthy status, then fix the value and confirm the fleet returns to healthy. |
| **Concepts** | Rollout/deployment health visibility; configuration drift and configuration errors; diagnosing failures from fleet health signals; closing the loop from "broken" to "healthy" using only Fleet Control. |
| **Time** | 15 min (diagnostic reasoning with the least scaffolding in the track, reflecting its advanced difficulty) |
| **Difficulty** | advanced |

**Total estimated time:** ~60 minutes across 5 challenges (15 + 8 + 10 + 12 + 15), basic → advanced. All challenges fall within a 5-15 minute band (no more than 1.9x variation between shortest and longest); Challenge 1 sits at the top of that band specifically because of its documented install/reconciliation wait, not because it needs more scaffolding than Challenge 5.

## Confirmed decisions

The following were open items during planning and have since been confirmed by the user — treat these as settled for challenge-planning and generation, not open questions to re-litigate:

1. **Account/credential model:** a single shared New Relic account is used across all concurrent learner sessions. Setup scripts must derive a unique fleet name per session (e.g. `fleet-control-lab-${INSTRUQT_PARTICIPANT_ID}`) so concurrent learners don't collide within that shared account.
2. **Real secret values:** the license key and system identity credentials (`organizationId`, `clientId`, `clientSecret`) will be supplied by the user at generation time, not before. Continue using placeholder secret names (`NEW_RELIC_LICENSE_KEY`, `NR_SYSTEM_IDENTITY_ORG_ID`, `NR_SYSTEM_IDENTITY_CLIENT_ID`, `NR_SYSTEM_IDENTITY_CLIENT_SECRET`) in config.yml and scripts until those real values are provided.
3. **Scope:** confirmed Kubernetes-only — no hands-on Linux/Windows host management. Linux/Windows fleet types may still be mentioned conceptually in assignment text, but no host provisioning or check/solve scripts should target them.
4. **Source freshness:** accepted as-is. The plan's terminology, features, and YAML examples come from a September 2026 fetch of New Relic's docs. A few very specific details flagged as unverified in that fetch (an exact health-endpoint port, a dated log filename, a minute-by-minute Windows install timeline) are not used anywhere in this Kubernetes-focused plan, so this is not a blocker.
