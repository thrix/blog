---
title: "Silencing Alerts During OpenShift Cluster Upgrades with Prometheus and Alertmanager"
date: 2026-03-09
draft: false
tags: ["openshift", "prometheus", "alertmanager", "monitoring", "alerts"]
summary: "How to detect OpenShift cluster upgrades, extend the detection window to cover the recovery period, and use Alertmanager inhibit rules to suppress noisy alerts during upgrades."
---

If you run workloads on OpenShift and monitor them with Prometheus and
Alertmanager, you've likely dealt with a flood of alerts every time the cluster
gets upgraded. Pods restart, nodes drain, services go temporarily unavailable —
and your on-call gets paged for things that are expected and transient.

In this post I'll walk through how we handle this in our monitoring stack: detecting
when an upgrade is happening, extending that detection window to cover the
recovery period, and using Alertmanager inhibit rules to suppress alerts that
would otherwise fire during this expected disruption.

## Our monitoring stack

We run a fairly standard Prometheus and Alertmanager setup on OpenShift. Prometheus
scrapes metrics from the cluster and our applications, evaluates alerting rules,
and sends firing alerts to Alertmanager. Alertmanager handles routing,
deduplication, grouping, and — crucially for this post — inhibition of alerts.

Our alerting rules and Alertmanager configuration are managed as code in a Git
repository and applied through CI/CD.

## Detecting a cluster upgrade

OpenShift exposes cluster version information through the `cluster_version`
metric, exposed by the
[Cluster Version Operator](https://github.com/openshift/cluster-version-operator).
When an upgrade is in progress, a time series with `type="updating"` is emitted.

A simple Prometheus alerting rule to detect an ongoing upgrade:

```yaml
- alert: ClusterUpgradeInProgress
  expr: sum(cluster_version{type="updating"}) > 0
  labels:
    severity: info
  annotations:
    summary: "OpenShift cluster upgrade is ongoing."
    description: |
      The OpenShift cluster is being upgraded. Pods will be migrated
      from upgraded nodes and restarted.
```

This fires once Prometheus evaluates the expression and finds the upgrade
metric active, and resolves once `cluster_version` no longer reports
`type="updating"`.

## The problem: the aftermath

The alert above works well for the upgrade itself, but services don't
recover instantly once the upgrade finishes. Pods need time to restart, caches
need to warm up, and various health checks may lag behind. Depending on your
workloads, it can take 30 minutes to an hour before everything is fully stable
again.

During this recovery window, you'll still see alerts firing for conditions that
are a direct consequence of the upgrade — even though the upgrade metric has
already gone back to zero.

## Extending the detection window

Prometheus supports
[subqueries](https://prometheus.io/docs/prometheus/latest/querying/basics/#subquery),
which let you evaluate an instant query over a time range. We can use
`max_over_time` to create an alert that stays firing for a configurable period
after the upgrade completes:

```yaml
- alert: ClusterUpgradeRecovery
  expr: max_over_time(sum(cluster_version{type="updating"})[1h:]) > 0
  labels:
    severity: info
  annotations:
    summary: "OpenShift cluster upgrade is ongoing or recently completed."
    description: |
      The cluster is being upgraded or was upgraded within the last hour.
      Services may still be recovering.
```

The expression `max_over_time(sum(cluster_version{type="updating"})[1h:])` looks
at the maximum value of the upgrading metric over the last 1 hour. If the cluster
was upgrading at any point during that window, the max will be greater than zero,
and the alert will keep firing.

The `[1h:]` part is the subquery syntax — `1h` is the lookback range and the
omitted resolution (after the colon) defaults to the global evaluation interval. You can
adjust the `1h` to whatever recovery window makes sense for your environment.

This gives us two distinct alerts:

- **ClusterUpgradeInProgress** — fires only while the upgrade is actively
  happening. Use this when you need to know the upgrade is ongoing right now.
- **ClusterUpgradeRecovery** — fires during the upgrade and for 1 hour after.
  Use this for suppressing alerts that may fire during the recovery period.

## Alertmanager inhibit rules

Alertmanager
[inhibit rules](https://prometheus.io/docs/alerting/latest/configuration/#inhibit_rule)
let you prevent certain alerts from being routed to receivers when other alerts
are firing. The concept is simple: if a "source" alert is active, any matching
"target" alerts are suppressed. The inhibited alerts are still visible in the
Alertmanager UI marked as "inhibited", which is useful for debugging.

An inhibit rule has three key parts:

- **source_matchers** — conditions that identify the source alert (the one that,
  when firing, causes inhibition)
- **target_matchers** — conditions that identify the target alerts (the ones to
  suppress)
- **equal** (optional) — labels that must have the same value in both source and
  target for the inhibition to apply

Note: older Alertmanager versions used `source_match`/`target_match` (and their
`_re` variants). These still work but are deprecated since Alertmanager 0.22 in
favor of `source_matchers`/`target_matchers`, which use PromQL-style label
matching syntax.

Here's a basic example:

```yaml
inhibit_rules:
  # Suppress InstanceDown alerts for services during cluster upgrade recovery
  - target_matchers:
      - alertname = InstanceDown
      - instance =~ ".*myservice.*"
    source_matchers:
      - alertname = ClusterUpgradeRecovery
```

When `ClusterUpgradeRecovery` is firing, any `InstanceDown` alert whose
`instance` label matches `.*myservice.*` will be suppressed. No pages, no
noise.

## Putting it all together

Here's a complete example. Say you have a service that processes a message queue, and
it tends to appear stuck during and shortly after cluster upgrades because its
workers need time to reconnect and catch up.

First, the Prometheus alerting rules:

```yaml
groups:
  - name: cluster
    rules:
      # Fires only during active upgrade
      - alert: ClusterUpgradeInProgress
        expr: sum(cluster_version{type="updating"}) > 0
        labels:
          severity: info
        annotations:
          summary: "OpenShift cluster upgrade is ongoing."

      # Fires during upgrade + 1 hour recovery window
      - alert: ClusterUpgradeRecovery
        expr: max_over_time(sum(cluster_version{type="updating"})[1h:]) > 0
        labels:
          severity: info
        annotations:
          summary: "Cluster upgrade ongoing or recently completed."

  - name: myservice
    rules:
      - alert: StuckQueueProcessing
        expr: pending_messages > 50
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Message queue processing appears stuck."
```

Then the Alertmanager inhibit rules:

```yaml
inhibit_rules:
  # Node status can be unhealthy during upgrade
  - target_matchers:
      - alertname = ClusterNodeStatus
    source_matchers:
      - alertname = ClusterUpgradeInProgress

  # Queue processing needs extra recovery time after upgrade
  - target_matchers:
      - alertname = StuckQueueProcessing
    source_matchers:
      - alertname = ClusterUpgradeRecovery

  # Instances go down during upgrade
  - target_matchers:
      - alertname = InstanceDown
    source_matchers:
      - alertname = ClusterUpgradeInProgress
```

Notice how different alerts use different source alerts depending on their
recovery characteristics:

- `ClusterNodeStatus` and `InstanceDown` use `ClusterUpgradeInProgress` directly
  because nodes and instances typically recover shortly after the upgrade
  completes.
- `StuckQueueProcessing` uses `ClusterUpgradeRecovery` because queue processing
  needs significantly more time to reconnect and catch up after the upgrade.

## Tips

A few things to keep in mind when setting up inhibit rules:

- **Be specific with target matching.** Use exact matchers (`=`) for alert names
  or regex matchers (`=~`) for patterns. Overly broad rules can mask real
  problems.
- **Use the `equal` field** when you need to scope inhibition to specific
  instances or clusters. For example, if you monitor multiple clusters, add
  `equal: [cluster]` so an upgrade on cluster A doesn't silence alerts from
  cluster B.
- **Choose the right recovery window.** Start with 1 hour and adjust based on
  how long your services actually take to stabilize. Check your alert history
  after a few upgrades to calibrate.
- **Keep the original upgrade alert.** Having both `ClusterUpgradeInProgress`
  and `ClusterUpgradeRecovery` gives you flexibility — some alerts only need
  suppression during the upgrade itself, while others need the extended window.
- **Test your inhibit rules.** Use `amtool check-config` to validate your
  Alertmanager configuration syntax. Then, send test alerts to a running
  Alertmanager instance to confirm inhibition works as expected before relying
  on it during a real upgrade.

## Conclusion

Cluster upgrades are routine but disruptive. Rather than training your team to
ignore alerts during upgrades — which risks missing real issues — use the
tooling that Prometheus and Alertmanager already provide. Detect the upgrade
with metrics, extend the window with subqueries, and suppress the noise with
inhibit rules. Your on-call will thank you.
