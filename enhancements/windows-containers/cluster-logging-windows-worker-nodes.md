---
title: openshift-windows-worker-node-and-container-logging
authors:
  - "@ravisantoshgudimetla"
  - "@aravindhp"
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
last-updated: 2021-01-21
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
and pods running in an OpenShift cluster.

## Motivation

Logging is a critical component for identifying issues with nodes, containers
running on the nodes. The main motivation behind this enhancement is to provide
a centralized logging solution for the Windows nodes that will help gather useful
logs from different components for debugging issues and monitoring cluster activity.

### Goals

As part of this enhancement, we plan to do the following:
* Deploy log collectors on Windows nodes.
* Collect node and pod logs.
* Upgrade log collectors deployed through WMCO.
* Forward logs and events from nodes and pods to log store.

### Non-Goals

As part of this enhancement, we do not plan to support:
* Development or customization of log collectors.
  * We plan to use the Microsoft provided Windows Fluentd binary and the Fluentd
    logging driver that enables Windows containers to read logs from various sources
    using Fluentd input plugins and redirect them to various destinations using the 
    Fluentd output plugins.


## Proposal

The main idea here is to deploy Fluentd as a log collection agent onto Windows 
nodes only after cluster logging operator is deployed on the OpenShift cluster. 
Fluentd has support for Windows logging. We plan to leverage it and run Fluentd 
as a Windows service.


### Implementation Details

The td-agent MSI installer will be added to the WMCO image payload.
Once both WMCO and the Cluster Logging Operator are deployed on the cluster,
following steps are required to configure logging:
 * Install td-agent.
    * td-agent is a stable distribution of Fluentd.
 * Run td-agent's fluentdwinsvc as a Windows service.
    * fluentdwinsvc is permanently registered as a Windows service by the .msi
      installer in the latest version of td-agent.
 * Install the required Fluentd input and output plugins onto Windows nodes.
 * Configure the container runtime to use Fluentd as the logging driver.
 * Configure td-agent to collect logs from various input sources and forward 
   them to elasticsearch data storage. 

### Justification

* We cannot deploy fluentd as a container on Windows nodes in the way it is 
  deployed on linux nodes by Cluster Logging Operator due to the support 
  reasons for Windows containers mentioned in the [WMCO enhancement](https://github.com/openshift/enhancements/blob/master/enhancements/windows-containers/windows-machine-config-operator.md#justification).
* The reason for using the .msi installer package is to have the ability
  to run Fluentd as a Windows service. This will allow Windows to manage the 
  process and ensure it is always running for log collection.
  
 
### Risks and Mitigations

The risks involved with this proposal are:
* The current proposal is dependent on the Microsoft provided td-agent .msi installer
  binary for features, support with logging on Windows nodes.

### Test Plan

We plan to add e2e tests to ensure:

* Fluentd is running as a Windows service on Windows nodes.
* Windows nodes are forwarding logs properly to elasticsearch.

### Graduation Criteria

This enhancement will start as GA.

### Upgrade / Downgrade Strategy

We will support upgrade to the Fluentd component, 
compatible with the release versions of WMCO. 

## Implementation History

Intial implementation includes a [ansible playbook](https://github.com/openshift/windows-machine-config-bootstrapper/tree/0cfab036cefbe4d658b160d9ae80289ce1e1249f/tools/ansible/tasks/logging) approach to install Fluentd
and the required plugins onto the Windows nodes.

## Drawbacks

The inability to deploy Fluentd pods on the Windows nodes adds
to the node configuration steps that includes configuring and managing
Fluentd service for each Windows node.

## Alternatives

An alternative approach would be to modify the cluster logging operator to
manage Fluentd service on the Windows nodes. 
We are deciding not to go ahead with this approach in the current timeframe
since the fluentd pod cannot be deployed as a Daemonset on the Windows nodes,
this introduces many unknowns in this project.

