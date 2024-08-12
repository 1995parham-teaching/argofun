<h1 align="center">Fun with <a href="https://argoproj.github.io/">ArgoCD</a></h1>

<p align="center">
  <img alt="banner" src="./.github/img/banner.png" height="200px" />
</p>

## Introduction

ArgoCD offers various integration options in your pipeline. This introduction will focus on one such approach,
which involves storing Helm charts alongside your project and managing values in a separate repository.

## Why? 🤔

It's recommended to develop charts alongside your main project,
ensuring that they remain up-to-date with the latest changes and updates.

Storing values in a Git repository allows you to track changes to your configuration over time,
providing a more transparent and auditable approach compared to using GitLab/GitHub secrets,
which do not retain version history.

## From Git to Production 🚀

Each Helm chart-based project consists of two distinct components: the chart itself and its associated values.
A reasonable approach is to develop the chart within the project's repository, ensuring that the chart remains
up-to-date with changes to the codebase. This is because the chart is closely tied to the project's code and
should reflect any changes made to it.

Next, create a separate repository, which we'll refer to as the 'values repository',
that depends on the project's chart. This repository will contain the values that are used to configure the chart.

```yaml
apiVersion: v2
name: saf-consumer
description: Queue with NATS Jetstream to remove all the erlangs from cloud

version: "0.0.0" # please note that this version is required

dependencies:
  - name: saf-consumer
    version: "1.2.0" # chart version not the application version (but it is a good idea to always sync them)
    repository: "https://1995parham.me/saf" # it must be a valid chart repository (it could be an oci registry too)
```

And then our values placed in the values' repository:

```yaml
saf-consumer: # dependent chart name
  hello: 123
```

Once the chart and values repository are set up, you can create an application in ArgoCD that utilizes this chart.
ArgoCD allows you to manage applications and their dependencies in a centralized manner.
Within ArgoCD, you can create projects, which serve as a container for related applications.
Under each project, you can define multiple applications, each with its own configuration and dependencies.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: saf-consumer
spec:
  destination:
    namespace: nats-testing
    server: https://kubernetes.default.svc
  project: nats
  source:
    helm:
      valueFiles:
        - values.yaml
        - testing-teh1-okd.yaml
    path: saf-consumer
    repoURL: git@github.com:1995parham/saf-values
    targetRevision: HEAD
```

Also, we can create an application set to deploy multiple application at the same time **even with discovery**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: bootstrap-testing
spec:
  generators: # generate applications from `values.json` in github.com/1995parham/saf-values
    - git:
        repoURL: git@github.com:1995parham/saf-values
        revision: HEAD
        files:
          - path: "values.json"
  template:
    metadata:
      name: "{{ name }}"
    spec:
      destination:
        namespace: "{{ namespace.name }}"
        server: https://kubernetes.default.svc
      project: nats
      source:
        helm:
          valueFiles:
            - "./{{ namespace.type }}-{{ region }}-okd4.yaml"
        path: "{{ name }}"
        repoURL: git@github.com:1995parham/saf-values
        targetRevision: HEAD
```

And here the `values.json` which provide information about our ArgoCD applications that exists in our
values' repository:

```json
[
  {
    "name": "nats",
    "region": "teh1",
    "namespace": {
      "name": "nats-testing",
      "type": "testing"
    }
  },
  {
    "name": "saf-consumer",
    "region": "teh1",
    "namespace": {
      "name": "nats-testing",
      "type": "testing"
    }
  },
  {
    "name": "saf-producer",
    "region": "teh1",
    "namespace": {
      "name": "nats-testing",
      "type": "testing"
    }
  }
]
```

## Upgrade your images

When you deployed your application using ArgoCD, you can use [`argocd-image-updater`](https://argocd-image-updater.readthedocs.io/en/stable/)
to change your images automatically when a new tag pushed into your registry. Also, you can ask it to commit it for you
too.

Unlike ArgoCD itself, there is no need to deploy it cluster-wise, you can deploy it just for your target team with the
following helm chart:

`Chart.yaml`:

```yaml
apiVersion: v2
name: snapppay-argocd-image-updater
type: application
version: 0.0.0
sources:
  - https://artifacthub.io/packages/helm/argo/argocd-image-updater
dependencies:
  - name: argocd-image-updater
    version: 0.11.0
    repository: https://argoproj.github.io/argo-helm
```

`values.yaml`:

```yaml
argocd-image-updater:
  resources:
    limits:
      cpu: 100m
      memory: 512Mi
    requests:
      cpu: 10m
      memory: 128Mi
  extraArgs:
  - --interval=1m
  config:
    applicationsAPIKind: argocd
    argocd:
      insecure: "true"
      grpcWeb: "false"
      insecure: "true"
      plaintext: "true"
      serverAddress: spcld-argocd-user-server.user-argocd
      token: <TOKEN>
    logLevel: debug
    registries:
    - name: Internal OKD Registry
      api_url: https://image-registry.openshift-image-registry.svc:5000
      prefix: image-registry.openshift-image-registry.svc:5000
      ping: false
      insecure: true
      credentials: secret:user-argocd/cluster-registry-view#cred

```

you only need to have a static account in ArgoCD and generate a token for them:

```bash
argocd login --username admin-ci --password admin --grpc-web argocd.okd4.teh-1.snappcloud.io --grpc-web-root-path /grpc-api

argocd account generate-token
```
