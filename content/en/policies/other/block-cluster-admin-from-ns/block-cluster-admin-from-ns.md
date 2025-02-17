---
title: "Block a Cluster Admin/s from CREATE/UPDATE/DELETE any object in a Namespace"
category: other
version: 1.9.0
subject: Namespace, ClusterRole, User
policyType: "validate"
description: >
    In some cases we would want to block operations (CREATE/UPDATE/DELETE) of certain privileged users (i.e. cluster-admins), in a specific namespace. In this policy, Kyverno look for all user operations (`CREATE, UPDATE, DELETE`), on every object kind (Pod,Deployment,Route,Service,etc.), in the testnamespace namespace, and for the `clusterRole cluster-admin`. The `subject User testuser` is also mentioned so it won’t include all the cluster-admins in the cluster, but will be flexiable enough to apply only for a sub-group of the cluster-admins in the cluster.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other/block-cluster-admin-from-ns/block-cluster-admin-from-ns.yaml" target="-blank">/other/block-cluster-admin-from-ns/block-cluster-admin-from-ns.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-cluster-admin-from-ns
  annotations:
    policies.kyverno.io/title: Block a Cluster Admin/s from CREATE/UPDATE/DELETE any object in a Namespace
    policies.kyverno.io/category: other
    policies.kyverno.io/subject: Namespace, ClusterRole, User 
    policies.kyverno.io/minversion: 1.9.0
    policies.kyverno.io/description: >-
          In some cases we would want to block operations (CREATE/UPDATE/DELETE) of certain privileged users (i.e. cluster-admins), in a specific namespace.
          In this policy, Kyverno look for all user operations (`CREATE, UPDATE, DELETE`), on every object kind (Pod,Deployment,Route,Service,etc.), in the testnamespace namespace, and for the `clusterRole cluster-admin`. The `subject User testuser` is also mentioned so it won’t include all the cluster-admins in the cluster, but will be flexiable enough to apply only for a sub-group of the cluster-admins in the cluster.
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: block-cluster-admin-from-ns
    match:
      any:
      - resources:
          kinds:
          - "*"
          namespaces:
          - testnamespace
        clusterRoles:
        - cluster-admin
        subjects:
        - kind: User
          name: testuser
    validate:
      message: "The cluster-admin 'testuser' user cannot touch testnamespace Namespace."
      deny:
        conditions:
          any:
            - key: "{{request.operation || 'BACKGROUND'}}"
              operator: AnyIn
              value:
              - CREATE
              - UPDATE
              - DELETE

```
