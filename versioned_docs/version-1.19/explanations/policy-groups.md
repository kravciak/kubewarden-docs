---
sidebar_label: Policy Groups
sidebar_position: 21
title: Policy Groups
description: A description of Kubewarden policy groups
keywords: [kubewarden, policy groups, clusteradmissionpolicygroup, admissionpolicygroup]
doc-persona: [kubewarden-operator]
doc-type: [explanation]
doc-topic: [explanations, policy-group]
---

<head>
  <link rel="canonical" href="https://docs.kubewarden.io/explanations/policy-groups"/>
</head>

The policy group feature allows users to create complex policies by combining
simpler ones. It introduces two new Custom Resource Definitions
(CRDs):

- `AdmissionPolicyGroup`: For admission policies that apply to specific
  namespaces.
- `ClusterAdmissionPolicyGroup`: For admission policies that apply across the
  entire cluster.

These policy groups enable users to use existing policies, reducing the
need for custom policy creation and enhancing reusability. By avoiding
duplication of policy logic, users can simplify management and create custom
policies with a DSL-like configuration.

Policy groups enable the combined evaluation of
multiple policies using logical operators. This allows the definition of
complex logic. However, it is important to note that while ordinary policies
can include mutation logic to modify resources during admission, policy groups
are limited to validation only.

Configuration for policy groups is similar to that of ordinary
policies. The difference is the addition of the `expression`,
`message`, and `policies` fields, as well as the declaration of context-aware
rules in a different location.

This is an example of a `ClusterAdmissionPolicyGroup` that we will use in
the next sections to explain the different fields:

<details>

<summary>
A `ClusterAdmissionPolicyGroup` that rejects Pods that use images with the `latest` tag,
unless the images are signed by two trusted parties: Alice and Bob.
</summary>

```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicyGroup # or AdmissionPolicyGroup
metadata:
  name: demo
spec:
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
        - UPDATE
  policies:
    signed_by_alice:
      module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.3.0
      settings:
        modifyImagesWithDigest: false
        signatures:
          - image: "*"
            pubKeys:
              - |
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEyg65hiNHt8FXTamzCn34IE3qMGcV
                yQz3gPlhoKq3yqa1GIofcgLjUZtcKlUSVAU2/S5gXqyDnsW6466Jx/ZVlg==
                -----END PUBLIC KEY-----
    signed_by_bob:
      module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.3.0
      settings:
        modifyImagesWithDigest: false
        signatures:
          - image: "*"
            pubKeys:
              - |
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEswA3Ec4w1ErOpeLPfCdkrh8jvk3X
                urm8ZrXi4S3an70k8bf1OlGnI/aHCcGleewHbBk1iByySMwr8BabchXGSg==
                -----END PUBLIC KEY-----
    reject_latest:
      module: registry://ghcr.io/kubewarden/policies/trusted-repos:v0.1.12
      settings:
        tags:
          reject:
            - latest
  expression: "reject_latest() || (signed_by_alice() && signed_by_bob())"
  message: "the image is using the latest tag or is not signed by Alice and Bob"
```

</details>

## Main configuration fields

This section covers the main configuration fields of a policy group.

### The `policies` attribute

The policies field is a map of ordinary policies. Kubewarden
policies are called by the policy group, to determine whether the resource under
evaluation is accepted or rejected. The definitions of these policies are a
simplified version of ordinary Kubewarden policies, containing only the
`module`, `settings` and `contextAwareResources` attributes. These
elements are necessary for the policies to function within a policy group.

Each policy of the group policy is identified by a unique name. For example,
the following snippet defines two policies: `signed_by_alice`, `signed_by_bob` and `reject_latest_tag`.

```yaml
policies:
  signed_by_alice:
    module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.2.8
    settings: {} # settings for the policy
  signed_by_bob:
    module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.2.8
    settings: {} # settings for the policy
  reject_latest_tag:
    module: ghcr.io/kubewarden/policies/trusted-repos-policy:v0.1.12
    settings: {} # settings for the policy
```

:::tip
The same policy can be included multiple times in the same policy group, with
different settings.
:::

### The `expression` attribute

The `expression` attribute contains a statement made of the policy
identifiers joined together by logical operators.

The evaluation of the `expression` statement must evaluate to a boolean value.

Each policy is represented as a function named after the identifier specified
inside of the `.spec.policies` map. The results produced
by the evaluation of the policies are then evaluated using the logical operators
provided by the user.

These are the supported operators:

- `&&`: used to perform `AND` operations
- `||`: used to perform `OR` operations
- `!`: used to perform `NOT` operations

Round brackets `( )` can be used to define evaluation priorities.

For example, given the following expression:

```yaml
reject_latest() || (signed_by_alice() && signed_by_bob())
```

The policy will reject workloads that have images using the `latest` tag, unless
these images are signed both by Alice and Bob.

### The `message` attribute and the response format

The `message` field specifies the message returned when the evaluation of the
`expression` results in a rejection. The message is included in the response,
together with the results of the individual policies evaluation.

Group Policies rely on the `warnings` attribute of the
[`AdmissionReview` response](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#response)
object to provide information about the evaluation results of the
policies that are part of the group.
These warnings are shown by multiple Kubernetes clients, including `kubectl`.

For example, this is the output produced when attempting to create a Pod with an image that
uses the `latest` tag and is signed only by Alice:

```shell
$ kubectl apply -f signed-pod.yml
Warning: signed_by_alice: allowed
Warning: signed_by_bob: rejected
Warning: reject_latest: rejected
Error from server: error when creating "signed-pod.yml": admission webhook "clusterwide-demo.kubewarden.admission" denied the request: the image is using the latest tag or is not signed by Alice and Bob
```

:::info
The policies that belong to the group are evaluated only
if necessary.

For example, given the following expression:

```yaml
reject_latest() || (signed_by_alice() && signed_by_bob())
```

The `signed_by_bob` and `signed_by_alice` policies are not evaluated when
the `reject_latest` policy returns `true`.

In the same way, the `signed_by_bob` policy is not evaluated if the `signed_by_alice`
and the `reject_latest` policies return `false`.

This avoids unnecessary evaluations of policies in the group and grants
fast responses to the admission requests.
:::

:::warning
The `warnings` attribute of the `AdmissionReview` response object are subject to
limitations.

Quoting the [official Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#response):

> Individual warning messages over 256 characters may be truncated by the API server
> before being returned to clients.
> If more than 4096 characters of warning messages are added (from all sources),
> additional warning messages are ignored.

Because of these limitations, the details about policy evaluation are not
provided as part of the `warnings` attribute of the `AdmissionReview` response.
:::

When a group policy performs a rejection, all the evaluation details of the
group policies are sent as part of the AdmissionResponse `.status.details.causes`.

The full details of a rejected admission request can be obtained by increasing the verbosity
level of `kubectl`:

```shell
kubectl -v4 apply -f signed-pod.yml
I0919 18:29:40.079805    4330 cert_rotation.go:137] Starting client certificate rotation controller
Warning: signed_by_alice: allowed
Warning: signed_by_bob: rejected
Warning: reject_latest: rejected
I0919 18:29:40.251332    4330 helpers.go:246] server response object: [{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "error when creating \"signed-pod.yml\": admission webhook \"clusterwide-demo.kubewarden.admission\" denied the request: the image is using the latest tag or is not signed by Alice and Bob",
  "details": {
    "causes": [
      {
        "message": "Resource signed is not accepted: verification of image testing.registry.svc.lan/busybox:latest failed: Host error: Callback evaluation failure: Image verification failed: missing signatures\nThe following constraints were not satisfied:\nkind: pubKey\nowner: null\nkey: |\n  -----BEGIN PUBLIC KEY-----\n  MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEswA3Ec4w1ErOpeLPfCdkrh8jvk3X\n  urm8ZrXi4S3an70k8bf1OlGnI/aHCcGleewHbBk1iByySMwr8BabchXGSg==\n  -----END PUBLIC KEY-----\nannotations: null\n",
        "field": "spec.policies.signed_by_bob"
      },
      {
        "message": "not allowed, reported errors: tags not allowed: latest",
        "field": "spec.policies.reject_latest"
      }
    ]
  },
  "code": 400
}]
Error from server: error when creating "signed-pod.yml": admission webhook "clusterwide-demo.kubewarden.admission" denied the request: the image is using the latest tag or is not signed by Alice and Bob
```

The full admission response is available in the logs of the Policy Server
when running in debug mode.
Moreover, the evaluation details are always part of the OpenTelemetry traces emitted by Policy Server.

## Context-Aware Policies

Another distinction between policy groups and ordinary policies is the location
where context-aware resource rules are defined. Each policy in a group
accepts an optional `contextAwareResources` field to specify the resources that
the policy is allowed to access during evaluation.

<details>

<summary>
An example of a policy group that makes use of a context-aware policy.
</summary>

```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicyGroup # or AdmissionPolicyGroup
metadata:
  name: demo-ctx-aware
spec:
  rules:
    - apiGroups:
        - ""
      apiVersions:
        - v1
      resources:
        - services
      operations:
        - CREATE
        - UPDATE
  polices:
    unique_service_selector:
      module: registry://ghcr.io/kubewarden/policies/unique-service-selector-policy:v0.1.0
      contextAwareResources:
        - apiVersion: v1
          kind: Service
      settings:
        app.kubernetes.io/name: MyApp
    owned_by_foo_team:
      module: registry://ghcr.io/kubewarden/policies/safe-annotations:v0.2.9
      settings:
        mandatory_annotations:
          - owner
        constrained_annotations:
          owner: "foo-team"
  expression: "unique_service_selector() || (!unique_service_selector() && owned_by_foo_team())"
  message: "the service selector is not unique or the service is not owned by the foo team"
```

</details>

In the previous example, the `unique_service_selector` policy is allowed to
access the `Service` resource. On the other hand, the `owned_by_foo_team`
has no access to Kubernetes resources.

## Settings Validation

When the policy server starts, it will validate the settings of both policy
groups and ordinary policies. However, policy groups undergo an additional
validation step to ensure that the expression is valid and evaluates to a
boolean value.

## Audit Scanner

Similar to the AdmissionPolicy and ClusterAdmissionPolicy CRDs, the
`backgroundAudit` field indicates if the policy group should be included
during [audit checks](../explanations/audit-scanner/audit-scanner.md).

## Policy Server

The `policies.yml` settings file is extended to include policy groups
alongside ordinary policies. As with ordinary policies, modules are
downloaded once. The same policy module is used in both a policy
group and an ordinary policy.
