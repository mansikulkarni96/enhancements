---
title: openshift-windows-worker-node-and-container-logging
authors:
  - "@mansikulkarni96"
reviewers:
  - "@crawford"
  - "@sdodson"
  - "@jcantrill"
  - "@richm" 
approvers:
  - "@crawford"
  - "@sdodson"
  - "@jcantrill"
  - "@richm"  
creation-date: 2019-09-10
last-updated: 2019-09-17
status: implementable
---

# OpenShift Windows Worker Node and Container Logging

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

The intent of this enhancement is to enable logging for Windows worker nodes 
and pods running in an OpenShift cluster

## Motivation

Logging is a critical component for identifying issues with nodes, containers
running on the nodes. The main motivation behind this enhancement is to provide
a logging solution for the Windows nodes that will help gather useful logs from
different components for debugging issues and monitoring cluster activity.

### Goals

As part of this enhancement, we plan to do the following:
* Deploy log collection infrastructure onto the Windows nodes.
* Collect node and pod logs.
* Upgrade log collection infrastructure.
* Forward logs and events from nodes and pods to log store.

### Non-Goals

As part of this enhancement, we do not plan to support:
* Logging for Windows containers.
* Customization of log collection infrastructure.
  * We plan to use the Microsoft provided Windows Fluentd binary and the Fluentd
    logging driver that enables Windows containers to redirect logs to various
    destinations using the Fluentd output plugins.


## Proposal

The main idea here is to deploy Fluentd as a log collection agent onto Windows 
nodes after cluster logging operator is deployed onto the OpenShift cluster. 
Fluentd has support for Windows logging. We plan to leverage it and run Fluentd 
as a Windows service.


### Implementation Details

Once WMCO is installed from the OperatorHub and Cluster Logging operator is deployed 
onto the cluster, the WMCO will perform the following steps to configure logging:

 * Download the td-agent .msi installer binary from a known location and 
      transfer it to the Windows node.
 * Run td-agent's fluentdwinsvc as a Windows service.
         * td-agent is a stable distribution of Fluentd, fluentdwinsvc
           is permanently registered as a Windows service by the msi installer.
 * Install the required Fluentd output plugins onto the Windows nodes.
 * Configure the container runtime to use Fluentd as the logging driver on Windows node.
 * Configure td-agent to forward logs to the elasticsearch data storage in a format
   understandable to elasticsearch. 
 * Kubelet on Windows nodes logs to a file. So, we also need to configure 
   fluentd's record transformer to ensure that logs collected from the kubelet log
   file are in required format.
          

### Justification

* The reason we want to use the .msi installer package is to have the ability
  to run Fluentd as a Windows service. This will allow Windows to manage the 
  process and ensure it is always running for log collection.
  
 
### Risks and Mitigations

The risks involved with this proposal are:
* Dependency on the Microsoft provided td-agent .msi installer for providing a logging solution.

### Test Plan

We plan to add e2e tests to ensure:

* Fluentd is running as a Windows service on Windows nodes.
* Windows nodes are forwarding logs properly to elasticsearch.

### Graduation Criteria

This enhancement will start as GA.

### Upgrade / Downgrade Strategy

We will support upgrade / downgrade to the Fluentd component, 
compatible with the release versions of WMCO. 

## Implementation History

We had initially implemented a [ansible playbook](https://github.com/openshift/windows-machine-config-bootstrapper/tree/0cfab036cefbe4d658b160d9ae80289ce1e1249f/tools/ansible/tasks/logging) approach to install Fluentd
and the required plugins onto the Windows nodes.

## Drawbacks

The applications running in Windows containers may not always log to stdout
which is what container runtimes expect. Microsoft is working on tool that
is capable of collecting application metrics from disparate sources like 
ETW(Event Tracing for Windows), Perf Counter, Custom application logs and
route them to stdout for a container but this work is not production ready.
Given this limitation, the container logging on Windows nodes may not be
as good as their linux counterparts, causing a degraded experience to 
customers.


## Alternatives

An alternative approach would be to modify the cluster logging operator to
manage Fluentd service on the Windows nodes. 
We are deciding not to go ahead with this approach in the current timeframe
since the fluentd DaemonSet pod cannot be deployed onto the Windows nodes,
this introduces many unknowns in this project.

