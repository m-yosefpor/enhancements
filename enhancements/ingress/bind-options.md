---
title: bind-options
authors:
  - "@m-yosefpor"
reviewers:
  -
approvers:
  -
creation-date: 2021-08-04
last-updated: 2021-08-04
status: implementable
see-also:
  - https://github.com/openshift/cluster-ingress-operator/issues/633
  - https://github.com/openshift/api/issues/964
replaces:
superseded-by:
---

# Ingress Bind Options

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement extends the IngressController API to allow the user to
specify bindOptions in HostNetwork strategy and enables them to run multiple
instances of ingresses on the same node in the HostNetwork strategy, with
mitigating port binding conflicts.

## Motivation

When using HostNetwork strategy for ingressControllers, the default 80, 443, 10443, 10444, 1936 ports on the host are binded on HAProxy for http, https, no_sni, sni and stats correspondingly. However, those ports might be occupied by other processes (such as another set of ingressControllers), which makes it impossible to run multiple set of ingressControllers on the same nodes with HostNetwork strategy. In OKD3.11, it was possible to listen on custom host ports via setting these env vars in Routers DeploymentConfig, however there is not any options to specify custom ports in `IngressController.operator.openshift.io/v1` object right now.

Having multiple sets of ingressControllers is useful for router sharding, routers with different policies (public, private), routers with different configuration options, etc. Running in HostNetwork is a strict requirement in some scenarios (e.g. environments with custom PBR rules). Also there might be some performance considerations to run routers with HostNetwork strategy.

Using custom nodeSelectors to ensure different ingressControllers runs on different nodes is not always feasible, when there are not many nodes in the cluster.

### Goals


### Goal

Enable cluster administrators to configure IngressControllers which use "HostNetwork" endpoint publishing strategy with the bindings for the following ports:
- HTTP
- HTTPS
- SNI
- NoSNI
- Stats

### Non-Goals

1. Enabling cluster administrators to configure custom ports on IngressControllers that use any endpoint publishing strategy other than "HostNetwork".

## Proposal

To enable cluster administrators to configure IngressControllers to configure bindOptions on IngressControllers that use the "HostNetwork" endpoint publishing strategies: the IngressController API is extended by adding an optional `BindOptions` field with type `*IngressControllerBindOptions` to the `HostNetworkStrategy` struct:

```go
// HostNetworkStrategy holds parameters for the HostNetwork endpoint publishing
// strategy.
type HostNetworkStrategy struct {
	// ...

	// bindOptions defines parameters for binding haproxy in ingress controller pods.
	// All fields are optional and will use their respective defaults if not set.
	// See specific bindOptions fields for more details.
	//
	//
	// Setting fields within bindOptions is generally not recommended. The
	// default values are suitable for most configurations.
	//
	// +optional
	BindOptions *IngressControllerBindOptions `json:"bindOptions,omitempty"`
}
```

`IngressControllerBindOptions` is the follwoing struct:

```go
// IngressControllerBindOptions specifies options for binding haproxy in ingress controller pods
type IngressControllerBindOptions struct {

	// httpPort defines the port number which HAProxy process binds for
	// http connections. Setting this field is generally not recommended. However in
	// HostNetwork strategy, default http 80 port might be occupied by other processess
	//
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30000
	// +kubebuilder:default:=80
	// +optional
	HTTPPort int32 `json:"httpPort,omitempty"`

	// httpsPort defines the port number which HAProxy process binds for
	// https connections. Setting this field is generally not recommended. However in
	// HostNetwork strategy, default https 443 port might be occupied by other processess
	//
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30000
	// +kubebuilder:default:=443
	// +optional
	HTTPSPort int32 `json:"httpsPort,omitempty"`

	// sniPort is for some internal front-end to back-end communication
	// This port can be anything you want as long as they are unique on the machine.
	// This port will not be exposed externally.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30000
	// +kubebuilder:default:=10444
	// +optional
	SNIPort int32 `json:"sniPort,omitempty"`

	// noSniPort is for some internal front-end to back-end communication
	// This port can be anything you want as long as they are unique on the machine.
	// This port will not be exposed externally.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30000
	// +kubebuilder:default:=10443
	// +optional
	NoSNIPort int32 `json:"noSniPort,omitempty"`

	// statsPort is the port number which HAProxy process binds
	// to expose statistics on it. Setting this field is generally not recommended.
	// However in HostNetwork strategy, default stats port 1936 might
	// be occupied by other processess
	//
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30000
	// +kubebuilder:default:=1936
	// +optional
	StatsPort int32 `json:"statsPort,omitempty"`
}
```

The following example configures two IngressControllers with HostNetwork strategy that can run on the same nodes without port conflicts:

```yaml
---
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default-1
  namespace: openshift-ingress-operator
spec:
  endpointPublishingStrategy:
    type: HostNetwork
---
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default-2
  namespace: openshift-ingress-operator
spec:
  endpointPublishingStrategy:
    type: HostNetwork
    hostNetwork:
      bindOptions:
        httpPort: 11080
        httpsPort: 11443
        sniPort: 11444
        noSniPort: 11445
        statsPort: 1937
```

### Validation

Specifying `spec.endpointPublishingStrategy.type: HostNetwork` and omitting
`spec.endpointPublishingStrategy.hostNetwork` or
`spec.endpointPublishingStrategy.hostNetwork.bindOptions` is valid and implies
the default behavior, which is to use the default ports to bind to. The API validates
that any value provided for the ports defined in
`spec.endpointPublishingStrategy.hostNetwork.bindOptions` is greater than `1`, and lower than `30000` (to avoid conflicts with 30000-32767 nodePort services range).


### User Stories

#### As a cluster administrator, I need to configure bind ports for my IngressController

To satisfy this use-case, the cluster administrator can set the
IngressController's `spec.endpointPublishingStrategy.hostNetwork.bindOptions`.


### Implementation Details

Implementing this enhancement requires changes in the following repositories:

* openshift/api
* openshift/cluster-ingress-operator

OpenShift Cluster Ingress Operator, creates a deployment for Router with environment variables for port bindings which OpenShift Router already respects: `ROUTER_SERVICE_HTTP_PORT`, `ROUTER_SERVICE_HTTPS_PORT`, `ROUTER_SERVICE_SNI_PORT`, `ROUTER_SERVICE_NO_SNI_PORT`, `STATS_PORT`.

### Risks and Mitigations


Users might define conflicting ports which would cause HAProxy process to fail at startup. A mitigation to this risk is to implement a validation feature in the reconciliation loop of the Cluster Ingress Operator to ensure all the ports defined in the `bindOptions` section are unique, and return an error with a meaningful message if there are conflicting ports.

Also if the underlying IngressController implementation were to change away from HAProxy to a different implementation, some values such as `sniPort` and `noSniPort` might not be needed and loose their meaning. Also the `statsPort` teminology might be different for other tools such as `Envoy`, which `metricPort` might be more meaningful in that case.


## Design Details

### Test Plan

The controller that manages the IngressController Deployment and related
resources has unit-test coverage; for this enhancement, the unit tests are
expanded to cover the additional functionality.

The operator has end-to-end tests; for this enhancement, the following test can
be added:

1. Create an IngressController that enables the `HostNetwork` endpoint publishing strategy type without specifying `spec.endpointPublishingStrategy.hostNetwork.bindOptions`.
2. Verify that the IngressController configures:
    - `ROUTER_SERVICE_HTTP_PORT=80`
    - `ROUTER_SERVICE_HTTPS_PORT=443`
    - `ROUTER_SERVICE_SNI_PORT=10444`
    - `ROUTER_SERVICE_NO_SNI_PORT=10443`
    - `STATS_PORT=1936`

3. Update the IngressController to specify `spec.endpointPublishingStrategy.hostNetwork.bindOptions`.
4. Verify that the IngressController updates the router deployment to specify the corresponding values for `ROUTER_SERVICE_HTTP_PORT`, `ROUTER_SERVICE_HTTPS_PORT`, `ROUTER_SERVICE_SNI_PORT`, `ROUTER_SERVICE_NO_SNI_PORT`, `STATS_PORT`.

### Graduation Criteria

N/A.

#### Dev Preview -> Tech Preview

N/A.  This feature will go directly to GA.

#### Tech Preview -> GA

N/A.  This feature will go directly to GA.

#### Removing a deprecated feature

N/A.  We do not plan to deprecate this feature.

### Upgrade / Downgrade Strategy

On upgrade, the default configuration remains in effect.

If a cluster administrator upgraded to 4.9, then use different ports for bindOptions, and then downgraded to 4.8, the downgrade would turn all the ingressController ports with HostNetwork strategy back to default ports.  The administrator would be responsible for making sure there are not any port conflicts when downgrading to OpenShift 4.8.


### Version Skew Strategy

N/A.

## Implementation History

This enhancement is being implemented in OpenShift 4.9.

In OpenShift 3.3 to 3.11 it was possible to use `ROUTER_SERVICE_HTTP_PORT`, `ROUTER_SERVICE_HTTPS_PORT`, `ROUTER_SERVICE_SNI_PORT`, `ROUTER_SERVICE_NO_SNI_PORT`, `STATS_PORT` environment variables.


## Alternatives

N/A
