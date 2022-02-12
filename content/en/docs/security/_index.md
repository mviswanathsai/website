---
title: "Security"
description: "Security Processes and Guidelines"
weight: 68
---

Kyverno serves an admission controller and is a critical component of the Kubernetes control-plane. It is important to properly secure and monitor Kyverno. This section provides guidance on securing Kyverno and the security processes for the Kyverno project.

## Disclosure Process

Security vulnerabilities are best handled swiftly and discretely with the goal of minimizing the total time users remain vulnerable to exploits.

If you find or suspect a vulnerability, please email the security group at kyverno-security@googlegroups.com with the following information:
- description of the problem
- precise and detailed steps (include screenshots) that created the problem
- the affected version(s)
- any known mitigations

The Kyverno security response team will send an initial acknowledgement of the disclosure in 3-5 working days. Once the vulnerability and mitigation are confirmed, the team will plan to release any necessary changes based on the severity and complexity. Additional details on the security policy and processes are available in the Kyverno [git repo](https://github.com/kyverno/kyverno/blob/main/SECURITY.md).

## Contact Us

To communicate with the Kyverno team, for any questions or discussions, use [Slack](https://slack.k8s.io/#kyverno) or [GitHub](https://github.com/kyverno/kyverno).


## Issues

All security related issues are labeled as `security` and can be viewed with this query:

  [https://github.com/kyverno/kyverno/labels/security](https://github.com/kyverno/kyverno/labels/security)


## Release Artifacts

The Kyverno container images are available at:

  https://github.com/orgs/kyverno/packages?repo_name=kyverno

With each release, the following artifacts are uploaded:
- checksums.txt
- kyverno-cli_v<version_number>_darwin_x86_64.tar.gz
- kyverno-cli_v<version_number>_linux_x86_64.tar.gz
- kyverno-cli_v<version_number>_windows_x86_64.zip
- Source code (zip)
- Source code (tar.gz)


## Verifying Kyverno Container Images

Kyverno container images are signed using Cosign. To verify the container image, download the organization public key (https://github.com/kyverno/kyverno/blob/main/cosign.pub) into a file named `cosign.pub` and then:

1. Install Cosign
2. Configure the Kyverno signature repository:

```sh
export COSIGN_REPOSITORY=ghcr.io/kyverno/signatures
```

3. Verify the image:

```sh
cosign verify -key cosign.pub ghcr.io/kyverno/kyverno:latest
```

If the container image was properly signed, the output should be similar to:

```sh
Verification for kyverno/kyverno:latest --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
[{"critical":{"identity":{"docker-reference":"ghcr.io/kyverno/kyverno"},"image":{"docker-manifest-digest":"sha256:a847df12e2c1cab19af9d1bb34e599cb56cf57639c7d5c958a4bb568c1dad8f6"},"type":"cosign container image signature"},"optional":null}]
```

All three Kyverno images can be verified.

## Fetching the SBOM for Kyverno

An SBOM (Software Bill of Materials) in CycloneDX JSON format is published for each Kyverno release. To download and verify the SBOM for a specific version, install Cosign and run:

```sh
cosign download sbom ghcr.io/kyverno/sbom:latest
```

To save the SBOM to a file, run the following command:

```sh
cosign download sbom ghcr.io/kyverno/sbom:latest > kyverno.sbom.json
```

## Security Scorecard

Kyverno uses [Scorecards by OSSF](https://github.com/ossf/scorecard) to maintain repository-wide security standards. The current OSSF/scorecard score for Kyverno can be found in this [tracker issue](https://github.com/kyverno/kyverno/issues/2617). The Kyverno team is committed to achieving and maintaining a high score. Contributions are welcome.

## Vulnerability Scan Reports

The Kyverno Helm Chart is available via the [ArtifactHub page](https://artifacthub.io/packages/helm/kyverno/kyverno) along with an auto-generated [Security Report](https://artifacthub.io/packages/helm/kyverno/kyverno?modal=security-report) generated by ArtifactHub for all the releases.


## Security Best Practices

The following sections discuss related best practices for Kyverno:

### Pod security

Kyverno pods are configured to follow security best practices:
* `runAsNonRoot` is set to `true`
* `privileged` is set to `false`
* `allowPrivilegeEscalation` is set to `false`
* `readOnlyRootFilesystem` is set to `true`
* all capabilities are dropped
* limits and quotas are configured
* liveness and readiness checks are configured

### RBAC

The Kyverno RBAC configurations are described in the [installation](/docs/installation/#roles-and-permissions) section.

Use the following command to view all Kyverno roles:
```sh
kubectl get clusterroles,roles -A | grep kyverno
```
It is important to limit Kyverno to the required permissions and audit changes in the RBAC roles and role bindings. In particular, the default `kyverno:view` and `kyverno:generate` roles can be customized to match your requirements.

### Networking

Kyverno network traffic is encrypted and should be restricted using network policies or similar constructs.

By default, a Kyverno installation does not configure network policies (see [GitHub issue](https://github.com/kyverno/kyverno/issues/2917)). The [Kyverno Helm chart](https://artifacthub.io/packages/helm/kyverno/kyverno) has a `networkPolicy.enabled` option to enable a network policy. 

Kyverno requires the following network communications to be allowed:
* ingress traffic to port 9443 from the API server
* ingress traffic to port 9443 from the host for health checks
* ingress traffic to port 8000 if metrics are collected by Prometheus or other metrics collectors.
* egress traffic to the API server if the [API Call](/docs/writing-policies/external-data-sources/#variables-from-kubernetes-api-server-calls) feature is used.
* egress (HTTPS) traffic to OCI registries if [image verification](/docs/writing-policies/verify-images/) policy rules are configured.

### Webhooks

Use the following command to view all Kyverno roles:
```sh
kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations | grep kyverno
```

Kyverno creates the following mutating webhook configurations:
- `kyverno-policy-mutating-webhook-cfg`: handles policy changes to index and cache policy sets.
- `kyverno-resource-mutating-webhook-cfg`: handles resource admission requests to apply matching Kyverno mutate policy rules.
- `kyverno-verify-mutating-webhook-cfg`: periodically tests Kyverno webhook configurations 

Kyverno creates the following validating webhook configurations:
- `kyverno-policy-validating-webhook-cfg`: validates Kyverno policies with checks that cannot be performed via schema validation
- `kyverno-resource-validating-webhook-cfg`: handles resource resource admission requests to apply matching Kyverno validate policy rules.

#### Webhook Failure Mode

Kyverno policies are configured to **fail-closed** by default. This setting can be tuned on a [per policy basis](/docs/writing-policies/policy-settings/). Kyverno uses the configured policy set to automatically configure webhooks.

#### Webhook authentication and encryption

By default, Kyverno automatically generates and manage TLS certificates used for authentication with the API server and encryption of network traffic. To use a custom CA, please refer to the details in the [installation section](/docs/installation/#certificate-management).

### Recommended policies

The Kyverno community manages a set of [sample policies](/policies/).

At a minimum, the [Pod Security Standards](/policies/pod-security/) and [best practices](/policies/?policytypes=Best%2520Practices) policy sets are recommended for use.

### Securing policies

Kyverno policies can be used to mutate and generate namespaced and cluster-wide resources. Hence, policies should be treated as critical resources and access to policies should be protected using RBAC.

## Threat Model

The [Kubernetes SIG Security](https://github.com/kubernetes/community/tree/master/sig-security) team has defined an [Admission Control Threat Model](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md). It is highly recommended that Kyverno administrators read and understand the threat model, and use it as a starting point to create their own threat model.

The sections below list each threat, mitigation, and provide Kyverno specific details. 

### Threat ID 1 - Attacker floods webhook with traffic preventing its operations

[Threat Model Link]((https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-1---attacker-floods-webhook-with-traffic-preventing-its-operations)) 

**Mitigation:**

* [Mitigation ID 2 - Webhook fails closed](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-2---webhook-fails-closed)

  Kyverno policies are configured **fail-closed** by default. This setting can be tuned on a [per policy basis](/docs/writing-policies/policy-settings/). Kyverno uses the configured policy set to automatically configure webhooks.


### Threat ID 2 - Attacker passes workloads which require complex processing causing timeouts

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model)

**Mitigations:**

* [Mitigation ID 2 - Webhook fails closed](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-2---webhook-fails-closed)

  Kyverno policies are configured **fail-closed** by default. This setting can be tuned on a [per policy basis](/docs/writing-policies/policy-settings/). Kyverno uses the configured policy set to automatically configure webhooks.

*  [Mitigation ID 3 - Webhook authenticates callers](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-3---webhook-authenticates-callers)

      By default, Kyverno generates a CA and X.509 certificates for the webhook registration. A custom CA and certificates can be used as discussed in the [installation guide](/docs/installation/#option-2-use-your-own-ca-signed-certificate). Currently, Kyverno does not authenticate the API server. A network policy can be used to restrict traffic to the Kyverno webhook port.


### Threat ID 3 - Attacker exploits misconfiguration of webhook to bypass

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-3---attacker-exploits-misconfiguration-of-webhook-to-bypass)


**Mitigation:**

*  [Mitigation ID 8 - Regular reviews of webhook configuration catch issues ](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-8---regular-reviews-of-webhook-configuration-catch-issues)

    Kyverno automatically generates webhook configurations based on the configured policy set. This ensures that webhooks are always updates and minimally configured.

### Threat ID 4 - Attacker has rights to delete or modify the k8s webhook object

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-4---attacker-has-rights-to-delete-or-modify-the-k8s-webhook-object)

**Mitigation:**

*  [Mitigation ID 1 - RBAC rights are strictly controlled](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-1---rbac-rights-are-strictly-controlled)

    Kyverno RBAC configurations are described in the [installation section](/docs/installation/#roles-and-permissions). The `kyverno:webhook` role is used by Kyverno to configure webhooks. It is important to limit Kyverno to the required permissions and audit changes in the RBAC roles and role bindings. In particular, the default `kyverno:view` and `kyverno:generate` roles can be customized to match your requirements.

### Threat ID 5 - Attacker gets access to valid credentials for the webhook

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-5---attacker-gets-access-to-valid-credentials-for-the-webhook)

**Mitigation:**

* [Mitigation ID 2 - Webhook fails closed](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-2---webhook-fails-closed)

  Kyverno policies are configured **fail-closed** by default. This setting can be tuned on a [per policy basis](/docs/writing-policies/policy-settings/). Kyverno uses the configured policy set to automatically configure webhooks.

### Threat ID 6 - Attacker gains access to a cluster admin credential

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-6---attacker-gains-access-to-a-cluster-admin-credential)

**Mitigation**

  **N/A**

### Threat ID 7 - Attacker sniffs traffic on the container network

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-7---attacker-sniffs-traffic-on-the-container-network)

**Mitigation**

* [Mitigation ID 4 - Webhook uses TLS encryption for all traffic](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-4---webhook-uses-tls-encryption-for-all-traffic)

  Kyverno uses HTTPS for all webhook traffic.

### Threat ID 8 - Attacker carries out a MITM attack on the webhook

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-8---attacker-carries-out-a-mitm-attack-on-the-webhook)

**Mitigation**

* [Mitigation ID 5 - Webhook mutual TLS authentication is used](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-5---webhook-mutual-tls-authentication-is-used)

     By default, Kyverno generates a CA and X.509 certificates for the webhook registration. A custom CA and certificates can be used as discussed in the [installation guide](/docs/installation/#option-2-use-your-own-ca-signed-certificate). Currently, Kyverno does not authenticate the API server. A network policy can be used to restrict traffic to the Kyverno webhook port.


### Threat ID 9 - Attacker steals traffic from the webhook via spoofing

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-9---attacker-steals-traffic-from-the-webhook-via-spoofing)

**Mitigation**

* [Mitigation ID 5 - Webhook mutual TLS authentication is used](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-5---webhook-mutual-tls-authentication-is-used)

     By default, Kyverno generates a CA and X.509 certificates for the webhook registration. A custom CA and certificates can be used as discussed in the [installation guide](/docs/installation/#option-2-use-your-own-ca-signed-certificate). Currently, Kyverno does not authenticate the API server. A network policy can be used to restrict traffic to the Kyverno webhook port.

### Threat ID 10 - Abusing a mutation rule to create a privileged container

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-10---abusing-a-mutation-rule-to-create-a-privileged-container)

**Mitigation**

* [Mitigation ID 6 - All rules are reviewed and tested](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#Mitigation-id-6---all-rules-are-reviewed-and-tested)

     Kyverno rules are Kubernetes resources written in YAML and managed by an OpenAPIv3 schema. This approach makes it easy to understand policy definitions and to apply policy-as-code best practices, like code reviews, to Kyverno policies. The [Kyverno CLI](/docs/kyverno-cli/) provides a `test` command for executing unit tests as part of a continious delivery pipeline. 

### Threat ID 11 - Attacker deploys workloads to namespaces that are exempt from admission control

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-11---attacker-deploys-workloads-to-namespaces-that-are-exempt-from-admission-control)

**Mitigation**

* [Mitigation ID 1 - RBAC rights are strictly controlled](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-1---rbac-rights-are-strictly-controlled)

     Kyverno RBAC configurations are described in the [installation section](/docs/installation/#roles-and-permissions). The `kyverno:webhook` role is used by Kyverno to configure webhooks. It is important to limit Kyverno to the required permissions and audit changes in the RBAC roles and role bindings. In particular, the default `kyverno:view` and `kyverno:generate` roles can be customized to match your requirements.

     Kyverno does not exempt any namespaces by default. It allows configuration of exempt namespaces via a [ConfigMap](https://kyverno.io/docs/installation/#configmap-flags).

### Threat ID 12 - Block rule can be bypassed due to missing match (e.g. missing initcontainers)

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-12---block-rule-can-be-bypassed-due-to-missing-match-eg-missing-initcontainers)

**Mitigation**

* [Mitigation ID 6 - All rules are reviewed and tested](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#Mitigation-id-6---all-rules-are-reviewed-and-tested)

     Kyverno rules are Kubernetes resources written in YAML and managed by an OpenAPIv3 schema. This approach makes it easy to understand policy definitions and to apply policy-as-code best practices, like code reviews, to Kyverno policies. The [Kyverno CLI](/docs/kyverno-cli/) provides a `test` command for executing unit tests as part of a continious delivery pipeline. 

### [Threat ID 13 - Attacker exploits bad string matching on a blocklist to bypass rules](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-13---attacker-exploits-bad-string-matching-on-a-blocklist-to-bypass-rules)

[Threat Model Link]()

**Mitigation**

* [Mitigation ID 6 - All rules are reviewed and tested](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#Mitigation-id-6---all-rules-are-reviewed-and-tested)

     Kyverno rules are Kubernetes resources written in YAML and managed by an OpenAPIv3 schema. This approach makes it easy to understand policy definitions and to apply policy-as-code best practices, like code reviews, to Kyverno policies. The [Kyverno CLI](/docs/kyverno-cli/) provides a `test` command for executing unit tests as part of a continious delivery pipeline. 

### Threat ID 14 - Attacker uses new/old features of the Kubernetes API which have no rules

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-14---attacker-uses-newold-features-of-the-kubernetes-api-which-have-no-rules)

**Mitigation**

* [Mitigation ID 6 - All rules are reviewed and tested](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#Mitigation-id-6---all-rules-are-reviewed-and-tested)

     Kyverno rules are Kubernetes resources written in YAML and managed by an OpenAPIv3 schema. This approach makes it easy to understand policy definitions and to apply policy-as-code best practices, like code reviews, to Kyverno policies. The [Kyverno CLI](/docs/kyverno-cli/) provides a `test` command for executing unit tests as part of a continious delivery pipeline.

### Threat ID 15 - Attacker deploys privileged container to node running Webhook controller

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-15---attacker-deploys-privileged-container-to-node-running-webhook-controller)

**Mitigation**

* [Mitigation ID 7 - Admission controller uses restrictive policies to prevent privileged workloads](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#Mitigation-id-6---all-rules-are-reviewed-and-tested)

    The Kyverno [policy library](/policies/) contains policies to restrict container privileges and restrict access to host resources. The pod security and best practices policies are highly recommended.


### Threat ID 16 - Attacker mounts a privileged node hostpath allowing modification of Webhook controller configuration

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-16---attacker-mounts-a-privileged-node-hostpath-allowing-modification-of-webhook-controller-configuration)

**Mitigation**

* [Mitigation ID 7 - Admission controller uses restrictive policies to prevent privileged workloads](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#Mitigation-id-6---all-rules-are-reviewed-and-tested)

    The Kyverno [policy library](/policies/) contains policies to restrict container privileges and restrict access to host resources. The pod security and best practices policies are highly recommended.

### Threat ID 17 - Attacker has privileged SSH access to cluster node running admission webhook

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-17---attacker-has-privileged-ssh-access-to-cluster-node-running-admission-webhook)

**Mitigation**

  **N/A**

### Threat ID 18 - Attacker uses policies to send confidential data from admission requests to external systems

[Threat Model Link](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#threat-id-18---attacker-uses-policies-to-send-confidential-data-from-admission-requests-to-external-systems)

**Mitigation**

* [Mitigation ID 9 - Strictly control external system access](https://github.com/kubernetes/sig-security/blob/main/sig-security-docs/papers/admission-control/kubernetes-admission-control-threat-model.md#mitigation-id-9---strictly-control-external-system-access)

    See [Networking](/docs/security/#networking) for details on securing networking communications for Kyverno.