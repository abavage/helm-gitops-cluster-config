
# ROSA HCP Cluster Configuration (Helm Values)

##  Overview
This repository serves as the `single source of truth for configuration` (Helm values) used to bootstrap and manage Red Hat OpenShift Service on AWS (ROSA) Hosted Control Plane (HCP) clusters.

This repository `contains configuration values only`, not the [Helm charts](https://github.com/) themselves.

## Bootstrap Process & Architecture
Cluster bootstrap process follows a GitOps-driven workflow, which is orchestrated by two primary tools:

* `Terraform:` Provisions the foundational ROSA HCP cluster and installs ArgoCD (OpenShift GitOps).
* `ArgoCD (OpenShift GitOps):` Takes over after installation to manage all cluster applications and add-ons.

## ArgoCD Configuration
ArgoCD is deployed to follow a `separation of concerns` pattern:

* `Helm Charts (The "What"):` The base [Helm charts](https://github.com/) are stored in a separate, dedicated chart repository.
* `Configuration (This Repo - The "How"):` This repository provides the cluster-specific `infrastructure.yaml` file.

ArgoCD is configured as a multi source `application` to pull the charts from the [charts](https://github.com) repository and apply the specific configurations from `this` repository. This drives all application deployments and ensures each cluster is configured correctly based on the values defined here.

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

## Example
The following values in `infrastructure.yaml` for the test-app helm chart will be passed in from here.

* `chart`: Helm chart to install - [test-app](https://github.com/)
* `namespace`: Namespace where the `testa-app` will be created. 
* `values`:
    * `image`: Image to use for the test-app deployment
    * `roleArn`: The AWS IAM Role the test-app deployment will leverage.
    * `region`: The AWS region where test-app will be installed.
    * `clustername`: Name of the cluster where the test-app will be deployed.
    * `awsSecretStore`: test-app uses  CRD's from the External Secrets Operator to source secrets from AWS Secrets Manager in a cloud native manner.
    * `awsSecretStoreProperty`: The key used within the AWS Secrets Manager secret. 


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
```

### View the value of the secret 
    * `aws secretsmanager list-secrets --query "SecretList[].Name"`
    * `aws secretsmanager get-secret-value --secret-id <secret_name> | jq '.SecretString'`



## Structue for namespaces 
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

The `<namespace>.yaml` file is a list of values that will be applied to the [namespaces helm chart](https://github.com/) by ArgoCD. 

ArgoCD makes use of the ApplicationSet object to apply the `<directory_name>/<file.yaml>`. The [ApplicationSet](https://github.com/) object is applied to the cluster by the [gitops-operator-bootstap](https://github.com/) helm chart.

It provides powerful generators (like the Git Directory Generator) that automatically detect new application or configurations as they are added to the repository, when directory and file names are unknown. In this configuration a new `<directory_name>/<file.yaml>` can be added to the repository and the ArgoCD applicationSet will apply the configuration to the cluster. The resultant configuration on the cluster will be a new namespace, networkpolicy, limitrange and resourcequota.

The [namespaces helm chart](https://github.com/) will apply the following core namespace level objects to each namespaces. These objects are classed as `Guard Rail` which limit a users ability to consume excessive resources and traverse the internal cluster network. Without these resources users `could` consume all the cpu and memory on the cluster and traverse the cluster network between namespaces. 

These objects are:
* [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
* [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
* [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
* [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)

Example configuration for `<directory_name>/<file.yaml>` which will drive the [namespaces helm chart](https://github.com/) 
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

## Adding a new Cluster
When a new cluster is required the following will need to known in advance. 

1. The region the cluster will be classed as `nonprod` or `prod`.
2. The unique cluster name.
3. The namespaces to be created by ArgoCD.
4. The AWS Account ID where the cluster will be installed `aws sts get-caller-identity | jq '.Account'`. 
5. The AWS region where cluster will be installed eg `ap-southeast-4` | `ap-southeast-2`.
6. The list of required helm charts and operators to be installed.

### Example
A new cluster is required to be installed with the following specifications answering the above questions.

1. The region is classed as a `nonprod` cluster
2. The unique cluster name is `rosadev10`
3. The list of required namespaces to be managed by ArgoCD:
    1. app-team-1
    2. app-team-2
    3. app-team-3
4. The aws account id is `abcdefghi99` as reported by `aws sts get-caller-identity | jq '.Account'`
5. The cluster will be installed in `ap-southeast-4`
6. This is a test cluster mininal helm charts and operators are required.
    1. [test-app](https://github.com/) helm chart
    2. [openshift-external-secrets-operator](https://github.com/) helm chart *
    3. [openshift-cluster-config](https://github.com/) helm chart *
    4. [openshift-efs](https://github.com/) helm chart *
    5. [openshift-logging](https://github.com/) helm chart *

    \* Core Cluster Configuration


### Directory Structure
With the above information defined the next step is setting up the directory structure that ArgoCD is expecting in git.

```
nonprod
├── rosadev10
│   ├── infrastructure.yaml
│   └── namespaces
│       ├── app-team-1
│       │   └── app-team-1.yaml
│       ├── app-team-2
│       │   └── app-team-2.yaml
│       ├── app-team-3
│       │   └── app-team-3.yaml
```

### infrastructure.yaml
Defines infrastructure components as an array of YAML objects. This specific, opinionated format is designed to be consumed by an ArgoCD Application, which iterates over each object in the array.

With the above cluster requirments specificed infrastructure.yaml can now be populated.


`Note`:
* Every `roleArn` needs to be updated to reflect the correct AWS Account ID and Cluster Name. 
* When every AWS IAM Role is created it's prefixed with the cluster name.
    * arn:aws:iam::`aws account id`:`cluster name`-rosa-efs-csi-role.   


```
infrastructure:
  - chart: test-app
    namespace: test-app
    values:
      image: registry.redhat.io/rhel10/httpd-24:latest
      roleArn: arn:aws:iam::abcdefghi99:role/rosadev10-external-secret-store-test-app
      region: ap-southeast-4
      clustername: rosadev10
      awsSecretStore: rosadev10-test-app-secrets
      awsSecretStoreProperty: rosa_api_token
  - chart: openshift-external-secrets
    namespace: external-secrets-operator
    values:
      channel: tech-preview-v0.1
      csv: external-secrets-operator.v0.1.0
  - chart: openshift-cluster-config
    namespace:
    values:
      custom_label:
        key: platform.cenitex.vic
        value: default
  - chart: openshift-efs
    namespace: openshift-cluster-csi-drivers
    values:
      csv: aws-efs-csi-driver-operator.v4.19.0-202510291015
      roleArn: arn:aws:iam::abcdefghi99:role/rosaprd01-rosa-efs-csi-role
  - chart: openshift-logging
    namespace: openshift-logging
    values:
      csv: cluster-logging.v6.3.1
```

### Namespaces 
ArgoCd will apply the values defined in `rosadev10/namespaces/app-team-1/app-team-1.yaml` from the [namespaces](https://github.com/) helm chart from the `ApplicationSet` configured in `openshift-gitops` $(`oc get applicationset -n openshift-gitops`).

The values can be tuned for each namespace individually as required by each app-team.

```
name: app-team-1
namespace: app-team-1
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
