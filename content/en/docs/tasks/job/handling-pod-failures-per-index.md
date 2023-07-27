---
title: Handling pod failures per index
content_type: task
min-kubernetes-server-version: v1.28
weight: 70
---

{{< feature-state for_k8s_version="v1.28" state="alpha" >}}

<!-- overview -->

This document shows you how to use the
[Backoff limit per index](/docs/concepts/workloads/controllers/job#backoff-limit-per-index),
to run Indexed {{<glossary_tooltip text="Jobs" term_id="job">}} where pod
failures are handled independently for all indexes.

This feature allows you to:
* complete execution of all indexes, despite some indexes failing,
* better utilize the computational resources by avoiding unnecessary retries of indexes.

## {{% heading "prerequisites" %}}

You should already be familiar with the basic use of [Job](/docs/concepts/workloads/controllers/job/),
and [Indexed Job](/docs/concepts/workloads/controllers/job#completion-mode) in
particular.

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

Ensure that the [feature gate](/docs/reference/command-line-tools-reference/feature-gates/)
`JobBackoffLimitPerIndex` is enabled in your cluster. For the
[scenario](#backoff-limit-per-index-with-pod-failure-policy) of using
the feature with pod failure policy also ensure tha `JobPodFailurePolicy` is
enabled in your cluster.

## Using backoff limit per index to execute all indexes

With the following example, you can learn how to use Backoff limit per index to
ensure all indexes execute despite failures in some of them.

First, create a Job based on the config:

{{< codenew file="/controllers/job-backoff-limit-per-index-execute-all.yaml" >}}

by running:

```sh
kubectl create -f job-backoff-limit-per-index-execute-all.yaml
```

The job runs a short while. After it is finished, check the status of the pods:

```sh
kubectl get pods -l job-name=job-backoff-limit-per-index-execute-all
```

which returns output similar to this:
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

Additionally inspect the Job status:

```sh
kubectl get jobs job-backoff-limit-per-index-execute-all -o yaml
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

Also, note that, the Job is marked failed, as there is at least one failed index,
but all indexes have completed their execution.

Additionally, you may want to use the `.spec.maxFailedIndex` to terminate the
Job execution as soon as the specified number of failed indexes is exceeded.

### Clean up

Delete the Job you created:

```sh
kubectl delete jobs/job-backoff-limit-per-index-execute-all
```

The cluster automatically cleans up the Pods.

## Using backoff limit per index with pod failure policy to avoid unnecessary retries within an index {#backoff-limit-per-index-with-pod-failure-policy}

With the following example, you can learn how to use backoff limit per index,
along with pod failure policy to avoid unnecessary retries within an index.

First, create a Job based on the config:

{{< codenew file="/controllers/job-backoff-limit-per-index-fail-index.yaml" >}}

by running:

```sh
kubectl create -f job-backoff-limit-per-index-fail-index.yaml
```

First, let's inspect the pods after the job is finished:

```sh
kubectl get pods -l job-name=job-backoff-limit-per-index-fail-index
```

Returns output similar to this:
```
k get pods
NAME                                             READY   STATUS      RESTARTS   AGE
job-backoff-limit-per-index-fail-index-0-znzqp   0/1     Completed   0          76s
job-backoff-limit-per-index-fail-index-1-6gvb9   0/1     Error       0          76s
job-backoff-limit-per-index-fail-index-1-jjxnr   0/1     Error       0          65s
job-backoff-limit-per-index-fail-index-2-x8wrd   0/1     Error       0          70s
job-backoff-limit-per-index-fail-index-3-cjbn7   0/1     Completed   0          66s
job-backoff-limit-per-index-fail-index-4-j6d2l   0/1     Completed   0          62s
job-backoff-limit-per-index-fail-index-5-t8wf5   0/1     Completed   0          62s
job-backoff-limit-per-index-fail-index-6-q6vfh   0/1     Completed   0          57s
job-backoff-limit-per-index-fail-index-7-qpbfj   0/1     Completed   0          56s
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
    failed: 3
    conditions:
    - message: Job has failed indexes
      reason: FailedIndexes
      status: "True"
      type: Failed
```

Here, index `2` was not retried, as the first pod failure matched the pod
failure policy rule with `FailIndex` action. At this point the index was marked
failed and further retries were prevented. Index `1` was retried once and marked
failed after exceeding the `.backoffLimitPerIndex` limit.

### Clean up

Delete the Job you created:

```sh
kubectl delete jobs/job-backoff-limit-per-index-fail-index
```

The cluster automatically cleans up the Pods.

### Cleaning up

Delete the Job you created:

```sh
kubectl delete jobs/job-pod-failure-policy-config-issue
```

The cluster automatically cleans up the Pods.

## Alternatives

In order to approximate the behavior of limiting the maximal number of
failures per index you could set a very high `.spec.backoffLimit`, and use an
external controller to monitor the failures per index. Then, once the allowed
failure count per index is exceeded, decrease the `.spec.backoffLimit` to 1.
While this approach would terminate the Job when backoff limit per index is
exceeded, it would not guarantee that all other indexes can finish.
