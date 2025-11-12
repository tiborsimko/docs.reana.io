# Configuring user quotas

Since [REANA 0.8.0](https://blog.reana.io/posts/2021/release-0.8.0/#cpu-and-disk-quota-accounting),
it is possible to configure user quotas to limit the usage of resources, such as CPU and disk storage.
This page describes how to enable quotas on your REANA cluster and how to set usage limits.
If you are looking for documentation from the user perspective, please check [this page](https://docs.reana.io/advanced-usage/user-quotas/).

## Enabling and disabling quotas

Quotas are enabled by default in REANA. If you would like to disable them, you can set the [`quota.enabled`](https://github.com/reanahub/reana/tree/master/helm/reana) Helm value to `false`.

REANA automatically creates two user quota resources for you:

- `cpu` to limit the amount of CPU time available to run workflows;
- `disk` to limit the total storage space in user workflow's workspaces.

In addition to enabling quotas, you might want to decide how frequently the quota resources usage should be tracked and updated.
This is done using two options:

- [`quota.periodic_update_policy`](https://github.com/reanahub/reana/tree/master/helm/reana) Helm value defining a policy where users' quota resources usage is updated periodically via a cron job. The value is a [cron expression](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#schedule-syntax) and it indicates how frequently the cron job should run. By default, the value is set to `"0 3 * * *"`, meaning that the quota usage updates are carried out daily at 03:00 in the morning.
- [`quota.workflow_termination_update_policy`](https://github.com/reanahub/reana/tree/master/helm/reana) Helm value defining a policy where users' quota resources usage is updated as soon as each workflow run terminates. This policy is not enabled by default because it could lead to slower workflow termination times when the REANA system runs many concurrent user workflows.

Here is an example of a mixed configuration where, in addition to nightly CPU and disk quota usage updates, the CPU usage is also updated immediately after each workflow run terminates:

```yaml
quota:
  enabled: true
  # update quota usage every day at 3AM
  periodic_update_policy: "0 3 * * *"
  # after workflow finishes or fails update its cpu usage
  workflow_termination_update_policy: "cpu"
```

## Setting default quota limits

When user quotas are enabled, no limit on the actual usage of resources by the users is enforced by default. You could however set a default quota limit that will apply to each user by means of setting the [`quota.default_disk_limit`](https://github.com/reanahub/reana/tree/master/helm/reana) and [`quota.default_cpu_limit`](https://github.com/reanahub/reana/tree/master/helm/reana) Helm values:

```yaml
quota:
  default_disk_limit: 10737418240  # bytes (10 GiB)
  default_cpu_limit: 36000000      # milliseconds (10 hours)
  ...
```

Individual users will be able to run as many workflows as they want, provided that the total CPU usage time stays under 10 hours and the total consumed disk space stays under 10 GiB.

!!! warning
    If the default quota values are changed, they will only be applied to newly created users. Users that already exist will not be affected by this change.

## Setting individual quota limits

In addition to setting the default quota limits for all users, you may want to set different quota usage limits for different specific groups of users.
This can be done via the `reana-admin` tool that is present in the `reana-server` pod or via the REST API.

### Using the `reana-admin` tool

Obtain the administrator access token:

```console
$ export REANA_ACCESS_TOKEN=$(kubectl get secret reana-admin-access-token -o json | jq -r '.data | map_values(@base64d) | .ADMIN_ACCESS_TOKEN')
```

You can now set a custom quota limit to selected users:

```console
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin quota-set -e john.doe@example.org -r disk -l 250000 --admin-access-token $REANA_ACCESS_TOKEN
Quota limit 250000 for 'disk (shared storage)' successfully set to users ('john.doe@example.org',).
```

You can learn more about the `quota-set` administrative command options by running `kubectl exec -i -t deployment/reana-server -- flask reana-admin quota-set --help`.

### Using the Quota management REST API

As of REANA 0.95 release series, you can also set quota limits for specific users using the Quota management REST API at `/api/quota`.

All Quota management REST API requests must include the `X-Quota-Management-Secret` header containing the configured secret.
This secret should be set in your Helm `values.yaml` file:

```yaml
components:
  reana_server:
    environment:
      REANA_QUOTA_MANAGEMENT_SECRET: "my_secret"
```

If the secret is unset or empty, the Quota management REST API is disabled.

To get the current CPU quota limit and usage for the user `john.doe@example.org`, run:

```console
$ curl --fail-with-body -v -H "X-Quota-Management-Secret: my_secret" \
       "https://reana.example.org/api/quota?email=john.doe@example.org&resource_type=cpu"
```

The response contains the current limit and usage for the given user and resource:

```json
{
  "limit": 36000000,
  "message": "OK",
  "usage": 12000000
}
```

To set a custom disk quota limit of 250 GiB for the user `john.doe@example.org`, run:

```console
$ curl --fail-with-body -v -H "X-Quota-Management-Secret: my_secret" \
       -H "Content-Type: application/json" \
       -X POST \
       -d '{"email": "john.doe@example.org", "resource_type": "disk", "limit": 250000000000}' \
       "https://reana.example.org/api/quota"
```

The response contains the new limit and the current usage for the given user and resource:

```json
{
  "limit": 250000000000,
  "message": "OK",
  "usage": 12000000
}
```
