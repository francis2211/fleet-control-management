---
slug: explore-fleet-control
type: challenge
title: Organize the fleet and read fleet health
teaser: Learn where health, deployment and configuration signals live in Fleet Control
notes:
- type: text
  contents: |-
    # Explore Fleet Control

    In this challenge, you're tasked with
    - Finding your cluster in the fleet's Managed Entities
    - Reading an agent's effective configuration
    - Cross-checking Fleet Control against the cluster
    - Understanding permissions, platform scope and fleet health
tabs:
- title: Terminal
  type: terminal
  hostname: workstation
  workdir: /root
- title: Fleet Control
  type: website
  url: https://fleet-control.${_SANDBOX_ID}.instruqt.io
difficulty: basic
timelimit: 480
---
This challenge creates nothing and changes nothing. It teaches you to *read* Fleet Control, so that the next challenge's first configuration change lands on a familiar screen.

🧪 Step 1: Open your fleet
=======================

In the **Fleet Control** tab go to **New Relic Control → Fleets** and open the fleet from the last challenge (`cat /root/fleet-lab-name.txt` if you need the name). On the **Summary** tab, every fleet has three components:

- **Active Deployment**
- **Agents & Configurations**
- **Managed Entities**

Find your cluster in the **Managed Entities** table. Its status may still be catching up. We'll check it in Step 5.

🧪 Step 2: Agents and effective configuration
=======================

Open the **Agents** tab. It lists every agent running on entities in the fleet, and Agent Control itself appears here as an agent type. Click the Agent Control entry for your cluster to open its **effective configuration**, and note the Helm chart version shown.

🧪 Step 3: Cross-check against the cluster
=======================

Run this in the terminal to read the version actually deployed:

```
~/check-agent-control-version.sh
```

It should match what Fleet Control just showed you. That is the point of the effective-configuration view: it shows what is really running, not a cached or intended value.

🧪 Step 4: Who can manage a fleet?
=======================

Creating or editing a fleet requires the **Organization Manager** role in your New Relic account. In the fleet list, open the `...` menu and find **Edit permissions** to see where this appears. You don't need to change anything.

🧪 Step 5: Platform scope and health
=======================

Kubernetes cluster management is **generally available**. Linux and Windows host management are in **public preview**. Look for this distinction on the Fleets page. It's why this track stays on Kubernetes.

Now return to your fleet's **Summary** tab. The Managed Entities table should show your cluster as healthy. If not, wait a couple of minutes and refresh. This is the same 5–10 minute connection window from the last challenge.

🏁 Step 6: Look at Deployments
=======================

Open the fleet's **Deployments** tab. It's empty: Agent Control is running, but nothing has been deployed to it yet. The next challenge changes that.

Press **Check** to finish.
