
# ROSA HCP Cluster Configuration (Helm Values)

##  Overview
This repository serves as the `single source of truth for configuration` (Helm values) used to bootstrap and manage Red Hat OpenShift Service on AWS (ROSA) Hosted Control Plane (HCP) clusters.

This repository `contains configuration values only`, not the [Helm charts](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm) themselves.

## Bootstrap Process & Architecture
Cluster bootstrap process follows a GitOps-driven workflow, which is orchestrated by two primary tools:

* `Terraform:` Provisions the foundational ROSA HCP cluster and installs ArgoCD (OpenShift GitOps).
* `ArgoCD (OpenShift GitOps):` Takes over after installation to manage all cluster applications and add-ons.

## ArgoCD Configuration
ArgoCD is deployed to follow a `separation of concerns` pattern:

* `Helm Charts (The "What"):` The base [Helm charts](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm) are stored in a separate, dedicated chart repository.
* `Configuration (This Repo - The "How"):` This repository provides the cluster-specific `infrastructure.yaml` file.

ArgoCD is configured as a multi source `application` to pull the charts from the [charts](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm) repository and apply the specific configurations from `this` repository. This drives all application deployments and ensures each cluster is configured correctly based on the values defined here.

## Structure
Each cluster is defined by region `nonprod or prod` and `unique cluster name`. An infrastructure.yaml file is in each `unique cluster name` directory which defines all values for the helm charts.

* `infrastructure.yaml`: Defines infrastructure components as an array of YAML objects. This specific, opinionated format is designed to be consumed by an ArgoCD Application, which iterates over each object in the array.
    * Each array is then applied to its matching named helm chart.


```
├── nonprod
│   ├── rosadev01
│   │   ├── infrastructure.yaml
│   │   └── namespaces
│   │       ├── team-a
│   │       │   └── team-a.yaml
│   │       ├── team-b
│   │       │   └── team-b.yaml
│   │       ├── team-c
│   │       │   └── team-c.yaml
│   │       └── team-d
│   │           └── team-d.yaml
│   └── rosadev02
│       ├── infrastructure.yaml
│       └── namespaces
│           ├── team-a
│           │   └── team-a.yaml
│           └── team-b
│               └── team-b.yaml
└── prod
    ├── rosaprd01
    │   ├── infrastructure.yaml
    │   └── namespaces
    │       ├── team-a
    │       │   └── team-a.yaml
    │       ├── team-b
    │       │   └── team-b.yaml
    │       ├── team-c
    │       │   └── team-c.yaml
    │       └── team-d
    │           └── team-d.yaml
    ├── rosaprd02
    │   ├── infrastructure.yaml
    │   └── namespaces
    │       ├── team-a
    │       │   └── team-a.yaml
    │       └── team-b
    │           └── team-b.yaml
    └── rosaprd03
        ├── infrastructure.yaml
        └── namespaces
            ├── team-a
            │   └── team-a.yaml
            └── team-b
                └── team-b.yaml

```


### infrastructure.yaml
```
infrastructure:
  - chart: test-app
    namespace: test-app
    values:
      image: registry.redhat.io/rhel10/httpd-24:latest
      roleArn: arn:aws:iam::012345678901:role/one-external-secret-store-test-app
      region: ap-southeast-2
      clustername: one
      awsSecretStore: one-test-app-secrets
      awsSecretStoreProperty: rosa_api_token
  - chart: openshift-logging
    namespace: openshift-logging
    values:
      csv: cluster-logging.v6.3.1
  - chart: openshift-aap
    namespace: aap
    values:
      csv: aap-operator.v2.5.0-0.1758147230
```

Each region and cluster has a namespaces directory which contain indvidual directories which represent namespaces to be created on the cluster. 
Each defined directory has a matching `<namespace>.yaml` file which contains the specific `namespace` object values applied to the namespace. 

```
nonprod
├── rosadev01
│   ├── infrastructure.yaml
│   └── namespaces
│       ├── team-a
│       │   └── team-a.yaml
│       ├── team-b
│       │   └── team-b.yaml
│       ├── team-c
│       │   └── team-c.yaml
│       └── team-d
│           └── team-d.yaml
```

The `<namespace>.yaml` file is a list of values that will be applied to the [namespaces helm chart](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm/tree/main/charts/namespaces) by ArgoCD. 

ArgoCD makes use of the ApplicationSet object to apply the `<directory_name>/<file.yaml>`. The [ApplicationSet](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm/blob/main/charts/gitops-operator-bootstrap/templates/applicationset.namespace.yaml) object is applied to the cluster by the [gitops-operator-bootstap](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm/tree/main/charts/gitops-operator-bootstrap) helm chart.

It provides powerful generators (like the Git Directory Generator) that automatically detect new application or configurations as they are added to the repository, when directory and file names are unknown. In this configuration a new `<directory_name>/<file.yaml>` can be added to the repository and the ArgoCD applicationSet will apply the configuration to the cluster. The resultant configuration on the cluster will be a new namespace, networkpolicy, limitrange and resourcequota.

The [namespaces helm chart](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm/tree/main/charts/namespaces) will apply the following core namespace level objects to each namespaces. These objects are classed as `Guard Rail` which limit a users ability to consume excessive resources and traverse the internal cluster network. Without these resources users `could` consume all the cpu and memory on the cluster and traverse the cluster network between namespaces. 

These objects are:
* [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
* [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
* [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
* [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)

Example configuration for `<directory_name>/<file.yaml>` which will drive the [namespaces helm chart](https://github.com/VG-CTX-StorageUnixServices/awsvicgovprd01-rosa-helm/tree/main/charts/namespaces) 
```
name: bob
namespace: bob
custom_label:
  key: platform.openshift.io
  value: default
resourcequota:
  limits_cpu: 10
  limits_memory: 8Gi
  requests_cpu: 8
  requests_memory: 4Gi
limitrange:
  container:
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: 1000m
      memory: 1024Mi
  pod:
    max:
      cpu: 1000m
      memory: 4096Mi
```
