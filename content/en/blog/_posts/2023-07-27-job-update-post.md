---
layout: blog
title: "Kubernetes 1.28: Updates to the Job API"
date: 2023-07-27 
slug: kubernetes-1-28-jobapi-update
---

**Authors:** Kevin Hannon (G-Research), Michał Woźniak (Google)

This blog discusses two features to improve Jobs for batch users: PodRecreationPolicy and JobBackoffLimitPerIndex.

These are two features requested from users of the Job API to enhance a user's experience.

## Pod Recreation Policy

### What problem does this solve?

Many common machine learning frameworks, such as Tensorflow and JAX, require unique pods per Index. Currently, if a pod enters a terminating state (due to preemption, eviction or other external factors), a replacement pod is created and immediately fail to start.

Having a replacement Pod before the previous one fully terminates can also cause problems in clusters with scarce resources or with tight budgets. These resources can be difficult to obtain so pods can take a long time to find resources and they may only be able to find nodes once the existing pods have been terminated. If cluster autoscaler is enabled, the replacement Pods might produce undesired scale ups.

On the other hand, if a replacement Pod is not immediately created, the Job status would show that the number of active pods doesn't match the desired parallelism. To provide better visibility, the job status can have a new field to track the number of Pods currently terminating.

This new field can also be used by queueing controllers, such as Kueue, to track the number of terminating pods to calculate quotas.

### How can I use it

This is an alpha feature, which means you have to enable the `JobPodReplacementPolicy`
[feature gate](/docs/reference/command-line-tools-reference/feature-gates/),
with the command line argument `--feature-gates=JobPodReplacementPolicy=true`
to the kube-apiserver.

```yaml
kind: Job
metadata:
  name: new
  ...
spec:
  podReplacementPolicy: Failed
  ...
```

`podReplacementPolicy` can take either `Failed` or `TerminatingOrFailed`.  In cases where `PodFailurePolicy` is set, you can only use `Failed`.

This feature enables two components in the Job controller: Adds a `terminating` field to the status and adds a new API field called `podReplacementPolicy`.

The Job controller uses `parallelism` field in the Job API to determine the number of pods that it is expects to be active (not finished).  If there is a mismatch of active pods and the pod has not finished, we would normally assume that the pod has failed and the Job controller would recreate the pod.  In cases where `Failed` is specified, the Job controller will wait for the pod to be fully terminated (`DeletionTimeStamp != nil`).

### How can I learn more?

- Read the KEP: [PodReplacementPolicy](https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/3939-allow-replacement-when-fully-terminated)

## Job Backoff Limit per Index

### What is it for?

It allows you to create [Indexed](/docs/concepts/workloads/controllers/job/#completion-mode)
Jobs for which the pod failures are counted and limited independently per index.
This allows you to:
* complete execution of all indexes, despite some indexes failing,
* better utilize the computational resources by avoiding unnecessary retries of indexes.

One use case you can think of is running integration tests where each index
corresponds to a testing suite. In that case, you may want to account for
possible flake tests, allowing for 1 or 2 retries per suite. Additionally, when
there are buggy suites, making some of the indexes failing consistently, you
would like to terminate retries for that indexes, yet allowing other suites
to complete.

### How to use it?

Just create an Indexed Job with the `.spec.backoffLimitPerIndex` field specified.
For more details, let take a look at the example.

#### Example

{{< note >}}
Ensure that the [feature gate](/docs/reference/command-line-tools-reference/feature-gates/)
`JobBackoffLimitPerIndex` is enabled in your cluster.
{{< note >}}

The following example demonstrates how to use this feature to make sure the
Job executes all indexes, and the number of failures is controller per index.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-backoff-limit-per-index-execute-all
spec:
  completions: 8
  parallelism: 2
  completionMode: Indexed
  backoffLimitPerIndex: 1
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: example
        image: python
        command:
        - python3
        - -c
        - |
          import os, sys, time
          id = int(os.environ.get("JOB_COMPLETION_INDEX"))
          if id == 1 or id == 2:
            sys.exit(1)
          time.sleep(1)
```

Now, inspect the pods after the job is finished:

```sh
kubectl get pods -l job-name=job-backoff-limit-per-index-execute-all
```

Returns output similar to this:
```
NAME                                              READY   STATUS      RESTARTS   AGE
job-backoff-limit-per-index-execute-all-0-b26vc   0/1     Completed   0          49s
job-backoff-limit-per-index-execute-all-1-6j5gd   0/1     Error       0          49s
job-backoff-limit-per-index-execute-all-1-6wd82   0/1     Error       0          37s
job-backoff-limit-per-index-execute-all-2-c66hg   0/1     Error       0          32s
job-backoff-limit-per-index-execute-all-2-nf982   0/1     Error       0          43s
job-backoff-limit-per-index-execute-all-3-cxmhf   0/1     Completed   0          33s
job-backoff-limit-per-index-execute-all-4-9q6kq   0/1     Completed   0          28s
job-backoff-limit-per-index-execute-all-5-z9hqf   0/1     Completed   0          28s
job-backoff-limit-per-index-execute-all-6-tbkr8   0/1     Completed   0          23s
job-backoff-limit-per-index-execute-all-7-hxjsq   0/1     Completed   0          22s
```

Additionally, let's take a look at the job status:

```sh
kubectl get jobs job-backoff-limit-per-index-fail-index -o yaml
```

Returns output similar to this:

```yaml
  status:
    completedIndexes: 0,3-7
    failedIndexes: 1,2
    succeeded: 6
    failed: 4
    conditions:
    - message: Job has failed indexes
      reason: FailedIndexes
      status: "True"
      type: Failed
```

Here, indexes `1`  and `2` were both retried once. After the second failure,
in each of them, the specified `.spec.backoffLimitPerIndex` was exceeded, so
the retries were stopped. For comparison, if the per-index backoff was disabled,
then the buggy indexes would retry until the global `backoffLimit` was exceeded,
and then the entire Job would be marked failed, before some of the higher
indexes are started.

### Getting Involved

These features were sponsored under the domain of SIG Apps.  Batch is actively being improved for Kubernetes users in the batch working group.  
Working groups are relatively short-lived initatives focused on specific goals.  In the case of Batch, the goal is to improve/support batch users and enhance the Job API for common use cases.  If that interests you, please join the working group either by subscriping to our [mailing list](https://groups.google.com/a/kubernetes.io/g/wg-batch) or on [Slack](https://kubernetes.slack.com/messages/wg-batch).

### Acknowledgments

As with any Kubernetes feature, multiple people contributed to getting this
done, from testing and filing bugs to reviewing code.

We would not have been able to achieve either of these features without Aldo Culquicondor (Google) providing excellent domain knowledge and expertise throughout the Kubernetes ecosystem.
