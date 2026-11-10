---
slug: remote-configure-infrastructure-agent
type: challenge
title: Remotely configure the Infrastructure agent
teaser: Push a configuration from Fleet Control and watch Flux apply it to the cluster
notes:
- type: text
  contents: |-
    # Remote configuration

    In this challenge, you're tasked with
    - Authoring a configuration for the New Relic Infrastructure agent
    - Enabling process metrics
    - Creating and starting a deployment
    - Verifying Flux applied it to the cluster
tabs:
- title: Terminal
  type: terminal
  hostname: workstation
  workdir: /root
- title: Fleet Control
  type: website
  url: https://fleet-control.${_SANDBOX_ID}.instruqt.io
difficulty: intermediate
timelimit: 720
---
🧠 Local vs. remote configuration
=======================

Agent Control began with only a *local* configuration: whatever was in `~/agent-control-values.yaml` at install time, which contains no agents. Because your cluster is an **assigned** managed entity, it now takes its configuration from Fleet Control. Today you give it something to receive.

🧪 Step 1: Author the configuration
=======================

1. In the terminal, get your lab name: `cat /root/fleet-lab-name.txt`
2. In the **Fleet Control** tab go to **New Relic Control → Configurations → Create configuration**.
3. Set **Managed entity type** to **Kubernetes Cluster** and **Agent type** to **New Relic Infrastructure**.
4. Name it `<your-lab-name>-infra`. Configurations are shared across the whole account, so a generic name would collide with other learners.
5. Click **Continue**, then **Copy** on the right-hand template pane, and paste it into the left-hand authoring pane.

🧪 Step 2: Enable process metrics
=======================

In the pasted template find the `chart_values` → `newrelic-infrastructure` block and set:

```
enableProcessMetrics: true
```

`chart_values` is a nested map keyed by the underlying chart name. Click **Save**.

🧪 Step 3: Create and start a deployment
=======================

On your fleet's page:

1. Click **Create a deployment** and give it a name.
2. Click **Add an agent** and select **New Relic Infrastructure** (agent type `newrelic/com.newrelic.infrastructure`). Pick the latest version and click **Next**.
3. Select the configuration you just saved, then **Add to deployment**.
4. Click **Save draft**, then **Start deployment** and confirm.

🧪 Step 4: Watch Flux reconcile it
=======================

```
kubectl get pods -n newrelic -w
```

Wait for Infrastructure agent pods to reach `Running`, then press Ctrl-C. This is Flux, the engine from Challenge 1, turning your remote configuration into a `HelmRelease`. Confirm it:

```
kubectl get helmrelease -A
```

🏁 Step 5: Close the loop
=======================

In Fleet Control, watch the deployment move from **IN PROGRESS** to **COMPLETED**. Then open the Infrastructure agent's effective configuration on the **Agents** tab and confirm `enableProcessMetrics: true` appears as *running* configuration.

Press **Check** to finish.
