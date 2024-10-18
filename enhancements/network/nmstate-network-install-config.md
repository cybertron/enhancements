---
title: nmstate-network-install-config
authors:
  - "@cybertron"
reviewers:
  - "@mkowalski"
  - "someone from installer"
approvers:
  - TBD
api-approvers:
  - TBD
creation-date: 2024-10-18
last-updated: 2024-10-18
tracking-link:
  - https://issues.redhat.com/browse/OPNET-592
see-also:
  - "/enhancements/network/configure-ovs-alternative.md"
  - "/enhancements/network/baremetal-ipi-network-configuration.md"
replaces:
  - NA
superseded-by:
  - NA
---

# Install-config interface for NMState configuration

## Summary

In the configure-ovs alternative enhancement we created a new mechanism
for deployers to configure host networking with NMState. This is similar,
but complementary, to the baremetal day 1 networking feature. In order to
provide the basic functionality in a timely fashion, the interface used
for the previous feature was machine-configs. We would now like to put a
formal install-config interface over that to improve the user experience.

## Motivation

Host networking is becoming increasingly important with the rise of
virtualization workloads in OpenShift. We are also anticipating that many
new virtualization customers will have limited experience with advanced
host network configuration and want to make the experience as smooth as
possible. While the existing machine-config interface is functional, it
requires manual creation of base64-encoded content and machine-config
manifests. All of this could be handled in the installer instead, which
would be a much better interface.

### User Stories
* As an OpenShift Virtualization user, I want full control over the
  host networking setup of my nodes so I can run VM workloads on nodes
  with only a single interface.

* As an OpenShift deployer, I want to create br-ex with NMState so it
  can be managed on day 2 with Kubernetes-NMState.

### Goals

Provide an install-config interface to the existing NMState functionality.

### Non-Goals

While this is a component of the multi-interface configuration feature,
anything beyond creating the install-config interface for NMState data
will be handled in a separate enhancement that is dependent on this one.

This also does not replace the existing networkConfig functionality for
the baremetal platform. Because this feature requires enough network
connectivity to pull Ignition, the two configuration mechanisms are
complementary. The baremetal feature can be used to configure, for
example, a static IP on the primary interface to allow Ignition to be
pulled, then this feature can be used to do full configuration on the
node.

## Proposal

Add a new field to the networking section of install-config that allows
users to provide their NMState config for cluster nodes. This will look
something like the networkConfig field for the baremetal platform, but
will be platform-agnostic.

### Workflow Description

A user will write an appropriate NMState config for their network environment
and include that in their install-config. The NMState YAML they provide will
then be base64-encoded and placed in a machine-config manifest that will be
included in the cluster configuration. From there, the flow will be the same
as described in the configure-ovs-alternative feature.

### API Extensions

There will be no additions to the API for this feature. It is purely an
install-config wrapper around existing functionality.

### Topology Considerations

#### Hypershift / Hosted Control Planes

This should be usable to configure host networking in any cluster deployed
using the installer. I do not anticipate needing anything Hypershift-specific
for it.

#### Standalone Clusters

Yes, standalone clusters are one of the primary motivators for this.

#### Single-node Deployments or MicroShift

It should have no effect on SNO. Once these configurations are applied
there is no ongoing cost in terms of CPU and memory.

I do not believe this would be especially useful for MicroShift either.
Since those devices are usually pre-configured, there wouldn't be much
need for deploy-time host network configuration from the installer.

### Implementation Details/Notes/Constraints

What are some important details that didn't come across above in the
**Proposal**? Go in to as much detail as necessary here. This might be
a good place to talk about core concepts and how they relate. While it is useful
to go into the details of the code changes required, it is not necessary to show
how the code will be rewritten in the enhancement.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

### Drawbacks

Exposing the full NMState API this way allows users a lot of flexibility in
how they configure their host networking, including bad configurations that
may not work well or at all. However, this is really no different from any
other existing host networking mechanism. We have not historically been
that prescriptive when it comes to host networking, and while that does
occasionally cause problems, it hasn't been a huge issue up to now and it's
unlikely this feature will make it worse.

All of this configuration can already be achieved by writing nmconnection
files to the nodes anyway. This is actually a slight improvement in that
NMState will validate syntax before applying changes and will roll back
changes that fail its built-in checks.

## Open Questions [optional]

None

## Test Plan

We should enable use of this feature in a metal CI job so it gets exercised
regularly.

## Graduation Criteria

This is part of the graduation process for the configure-ovs-alternative
feature. While we could GA it in its current state, feedback so far has been
that this would be much better.

### Dev Preview -> Tech Preview

NA

### Tech Preview -> GA

NA

### Removing a deprecated feature

NA

## Upgrade / Downgrade Strategy

Not relevant

## Version Skew Strategy

There will be no version skew at install time.

## Operational Aspects of API Extensions

Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
  Indicators) an administrator or support can use to determine the health of the API extensions

  Examples (metrics, alerts, operator conditions)
  - authentication-operator condition `APIServerDegraded=False`
  - authentication-operator condition `APIServerAvailable=True`
  - openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
  API availability)

  Examples:
  - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
  - Fails creation of ConfigMap in the system when the webhook is not available.
  - Adds a dependency on the SDN service network for all resources, risking API availability in case
    of SDN issues.
  - Expected use-cases require less than 1000 instances of the CRD, not impacting
    general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
  automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
  this enhancement)

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement.

## Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used
to highlight and record other possible approaches to delivering the
value proposed by an enhancement, including especially information
about why the alternative was not selected.

## Infrastructure Needed [optional]

None
