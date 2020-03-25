---
kep-number: 22
title: KEP-22: Diagnostics Bundle
short-desc: Automatic collection of diagnostics data for KUDO operators
authors:
  - "@mpereira"
owners:
  - "@mpereira"
  - "@gerred"
creation-date: 2020-01-24
last-updated: 2020-01-24
status: provisional
---

# [KEP-22: Diagnostics Bundle](https://github.com/kudobuilder/kudo/issues/1152)

## Table of contents

- [Summary](#summary)
- [Prior art, inspiration, resources](#prior-art-inspiration-resources)
  - [Diagnostics](#diagnostics)
- [Concepts](#concepts)
  - [Fault](#fault)
  - [Failure](#failure)
  - [Operator](#operator)
  - [Operator instance](#operator-instance)
  - [Application](#application)
  - [Operator developer](#operator-developer)
  - [Operator user](#operator-user)
  - [Diagnostic artifact](#diagnostic-artifact)
- [Goals](#goals)
  - [Functional](#functional)
  - [Non-functional](#non-functional)
- [Non-goals](#non-goals)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Operator user experience](#operator-user-experience)
  - [Operator developer experience](#operator-developer-experience)
- [Resources](#resources)
- [Implementation history](#implementation-history)

## Summary

Software will malfunction. When it does, data is needed so that it can be
diagnosed, dealt with in the short term, and fixed for the long term. This KEP
is about creating programs that will automatically collect data and store them
in an easily distributable format.

These programs must be easy to use, given that they will potentially be used in
times of stress where faults or failures have already occurred. Secondarily, but
still importantly, these programs should be easily extensible so that the
collection of data related to new fault types can be quickly implemented and
released.

Applications managed by KUDO operators are very high in the stack (simplified
below):

| Layer               | Concepts                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------ |
| Application         | (Cassandra's `nodetool status`, Kafka's consumer lag, Elasticsearch's cluster state, etc.) |
| Operator instance   | (KUDO plans, KUDO tasks, etc.)                                                             |
| KUDO                | (controller-manager, k8s events, logs, objects in kudo-system, etc.)                       |
| Kubernetes workload | (Pods, controllers, services, secrets, etc.)                                               |
| Kubernetes          | (Docker, kubelet, scheduler, etcd, cloud networking/storage, Prometheus metrics, etc.)     |
| Operating system    | (Linux, networking, file system, etc.)                                                     |
| Hardware            |                                                                                            |

These layers aren't completely disjoint. This KEP will mostly focus on:

- Application
- Operator instance
- KUDO
- Kubernetes workload

## Prior art, inspiration, resources

### Diagnostics

1.  [replicatedhq/troubleshoot](https://github.com/replicatedhq/troubleshoot)

    Does preflight checks, diagnostics collection, and diagnostics analysis for
    Kubernetes applications.

2.  [mesosphere/dcos-sdk-service-diagnostics](https://github.com/mesosphere/dcos-sdk-service-diagnostics/tree/master/python)

    Does diagnostics collection for
    [DC/OS SDK services](https://github.com/mesosphere/dcos-commons).

    Diagnostics artifacts collected:

    - Mesos-related (Mesos state)
    - SDK-related (Pod status, plans statuses, offers matching, service
      configurations)
    - Application-related (e.g., Apache Cassandra's
      [=nodetool](http://cassandra.apache.org/doc/latest/tools/nodetool/nodetool.html)=
      commands, Elasticsearch's
      [HTTP API](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)
      responses, etc.)

3.  [dcos/dcos-diagnostics](https://github.com/dcos/dcos-diagnostics)

    Does diagnostics collection for [DC/OS](https://dcos.io/) clusters.

4.  [mesosphere/bun](https://github.com/mesosphere/bun)

    Does diagnostics analysis for archives created with `dcos/dcos-diagnostics`.

    It is also important to notice that some applications have existing tooling
    for application-level diagnostics collection, either built by the supporting
    organizations behind the applications or the community. A few examples:

    - [Elasticsearch's support-diagnostics](https://github.com/elastic/support-diagnostics)
    - [Apache Kafka's System Tools](https://cwiki.apache.org/confluence/display/KAFKA/System+Tools)

## Concepts

### Fault

One component of the system deviating from its specification.

### Failure

The system as a whole stops providing the required service to the user.

### Operator

A KUDO-based
[Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/),
e.g. [kudo-cassandra](https://github.com/mesosphere/kudo-cassandra-operator),
[kudo-kafka](https://github.com/mesosphere/kudo-kafka-operator).

### Operator instance

An operator instance.

### Application

Underlying software that is managed by an operator instance, e.g., Apache
Cassandra, Apache Kafka, Elasticsearch, etc.

### Operator developer

Someone who builds and maintains operators.

### Operator user

Someone who installs and maintains operator instances.

### Diagnostic artifact

A file, network response, or command output that contains information that is
potentially helpful for operator users to diagnose faults with their operator
instances, and for operator users to provide to operator developers and/or
people who support operators.

## Goals

### Functional

- Collect "Kubernetes workload"-specific diagnostics artifacts related to an
  operator instance
- Collect KUDO-specific diagnostics artifacts related to an operator instance
- Collect application-specific diagnostics artifacts related to an operator
  instance
- Bundle all diagnostics artifacts into an archive

### Non-functional

1.  Provide an **easy** experience for operator users to collect diagnostic
    artifact archives
2.  Provide a **simple** experience for operator developers to extend and
    publish diagnostics artifacts collectors
3.  Be resilient to faults and failures. Collect as much diagnostics artifacts
    as possible and allow failed collections to be retried (idempotency) and
    incremented to (like `wget`'s `--continue` flag), in a way that collection
    is _resumable_
4.  Incorporate standard tools that are already provided by either organizations
    or the community behind applications as much as possible

## Non-goals

- Extensive collection of Kubernetes-related diagnostics artifacts
- At least not initially: collection of metrics from monitoring services (e.g.,
  Prometheus, Statsd, etc.).
- Automatic fixing of faults

## Requirements

- MUST create an archive with diagnostics artifacts related specifically to an
  operator instance
- MUST include application-related diagnostics artifacts in the archive
- MUST include instance-related diagnostics artifacts in the archive
- MUST include KUDO-related diagnostics artifacts in the archive
- MUST include "Kubernetes workload"-related diagnostics artifacts in the
  archive
- MUST accept parameters and work without interactive prompts
- SHOULD work in airgapped environments
- SHOULD report the versions of every component and tool, in the archive (e.g.,
  the version of the collector, the application version, the operator version,
  the KUDO version, the Kubernetes version, etc.)
- SHOULD follow Kubernetes' ecosystem conventions and best practices
- MUST be published as a static binary
- SHOULD make it possible to publish archive to cloud object storage (AWS S3,
  etc.)
- MUST follow SemVer

## Proposal

### Operator user experience

Three phases:

1.  Preflight check (before running operator instance)

    Checks:

    - Does the Kubernetes cluster have enough resources to install the operator
      with the given set of parameters?
    - Are there any KUDO-based checks that aren't passing?
    - Would it be possible to install the operator in that namespace given
      Kubernetes security policies and etc.?

    ```bash
    kubectl kudo diagnostics preflight %operator% --namespace=%namespace% \
            -p %parameter%=%value% \
            -p %parameter%=%value% \
            -p %parameter%=%value%
    ```

2.  Diagnostics collection (on a running operator instance)

    Collects diagnostics artifacts for:

    - Application
    - Operator instance
    - KUDO
    - Kubernetes-workload

    The output from diagnostics collection is an archive containing all
    diagnostics artifacts.

    ```bash
    kubectl kudo diagnostics collect --instance=%instance% --namespace=%namespace%
    ```

3.  Diagnostics analysis

    Is given an archive as input and provides a human-readable report as output
    containing explanations for any identified issues.

    ```bash
    kubectl kudo diagnostics analyze cassandra_diagnostics.zip
    ```

### Operator developer experience

TODO

## Resources

TODO

## Implementation history

TODO
