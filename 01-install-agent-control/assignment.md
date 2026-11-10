---
slug: install-agent-control
type: challenge
title: Install Agent Control and connect to Fleet Control
teaser: Install Agent Control on Kubernetes and connect the cluster to a fleet
notes:
- type: text
  contents: |-
    # Install Agent Control

    In this challenge, you're tasked with
    - Creating a fleet in Fleet Control
    - Adding the fleet ID to the Agent Control Helm values
    - Installing Agent Control on the k3s cluster with Helm
    - Confirming the cluster connects to Fleet Control
tabs:
- title: Terminal
  type: terminal
  hostname: workstation
  workdir: /root
- title: Fleet Control
  type: website
  url: https://fleet-control.${_SANDBOX_ID}.instruqt.io
difficulty: basic
timelimit: 900
---
🧠 What is Agent Control?
=======================

**Agent Control** is a lightweight agent supervisor that runs on your Kubernetes cluster and manages the lifecycle of New Relic and open-source instrumentation agents. It talks to **Fleet Control**, the centralized control plane in New Relic Control.

One rule shapes today's order: **you must create a fleet before installing Agent Control**, because the install is what assigns the cluster to that fleet.

🧪 Step 1: Create a fleet
=======================

1. In the terminal, print this session's unique lab name:

```
cat /root/fleet-lab-name.txt
```

2. In the **Fleet Control** tab, log in and go to **New Relic Control → Fleets → Create a fleet**.
3. Name the fleet exactly the lab name above, set **Type of Fleet** to **Kubernetes Cluster**, and click **Create Fleet**.

A **fleet** is a group of managed entities that share one instrumentation configuration. Once your cluster joins, it becomes an **assigned managed entity**. An **unassigned** one runs Agent Control but isn't in a fleet yet.

4. Open the new fleet, click **Metadata**, and copy its **Entity guid**. This is your fleet ID.

🧪 Step 2: Add the fleet ID to the Helm values
=======================

New Relic Control authenticates services with **system identities** (an `organizationId` plus a `clientId`/`clientSecret` pair). These are already provided on the workstation, so there is nothing to generate.

Open `~/agent-control-values.yaml`. It is pre-filled with your license key, system identity and cluster name. Replace `REPLACE_WITH_YOUR_FLEET_ID` under `agentControlDeployment.chartValues.config.fleet_control.fleet_id` with the Entity guid from Step 1.

```
nano ~/agent-control-values.yaml
```

🧪 Step 3: Install Agent Control
=======================

The New Relic Helm repo was already added for you. Confirm it:

```
helm repo list
```

Now install the `agent-control-bootstrap` chart:

```
helm upgrade --install agent-control-bootstrap newrelic/agent-control-bootstrap \
  -n newrelic-agent-control --create-namespace \
  -f ~/agent-control-values.yaml
```

This starts a short-lived Job that installs two more releases in the same namespace: `agent-control-deployment` (the supervisor itself) and `agent-control-cd`, an embedded **Flux** instance. Flux is the GitOps-style engine Agent Control uses to reconcile agents onto the cluster. You won't operate it directly, but you'll see its name again.

🧪 Step 4: Verify locally
=======================

```
helm list -n newrelic-agent-control
kubectl get pods -n newrelic-agent-control
```

All releases should be `deployed` and the pods `Running`.

> ⏳ It can take **5–10 minutes** after the pods are running before the cluster shows as healthy in Fleet Control. That's expected, not a stall.

🏁 Step 5: Confirm the connection in Fleet Control
=======================

In the **Fleet Control** tab, go to **Fleets**, open your fleet and look for your cluster in the **Managed Entities** table. Its status will read healthy once the connection completes. The next challenge explores this view in depth.

Press **Check** to finish.
