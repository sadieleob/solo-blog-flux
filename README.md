# Fluxcd and Gloo Edge

This repo was cloned from the [Fluxcd Project](https://github.com/fluxcd/flux2-kustomize-helm-example)

## Prerequisites

Request a Gloo Edge Enterprise [trial license](https://lp.solo.io/request-trial).

You will need a Kubernetes cluster version 1.16 or newer and kubectl version 1.18.
For a quick local test, you can use [Microk8s](https://www.solo.io/blog/a-local-development-kubernetes-environment-for-gloo-edge/).
Any other Kubernetes should work, in general the cluster needs to support Kubernetes ServiceType Loadbalancer and connectivity to those IPs.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS and Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains kubernetes application manifests, which are then customized per cluster.
- **infrastructure** dir contains common infrastructure services such as datastores, ingress controllers, etc. and Helm repository definitions. In this case [Gloo Edge Enterprise](https://docs.solo.io/gloo-edge/latest/) 
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   └── staging
├── infrastructure
│   ├── glooedge
│   ├── keycloak
│   └── sources
└── clusters
    └── staging
```

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and kubernetes manifest or Helm release definitions
- **apps/staging/** dir contains the staging values

```
├── apps
│   ├── base
│   │   └── petclinic
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       ├── service-app.yaml
│   │       ├── service-db.yaml
│   │       ├── statefulset-app.yaml
│   │       ├── statefulset-db.yaml
│   │       └── virtualservice.yaml
│   └── staging
│       ├── kustomization.yaml
│       ├── petclinic
│       │   ├── authconfig.yaml
│       │   └── kustomization.yaml
│       └── ratelimit.yaml
```

In **apps/base/petclinic/** dir we have a manifests with common values for both clusters:

```yaml
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: petclinic
  namespace: gloo-system
spec:
  virtualHost:
    domains:
      - '*'
    routes:
      - matchers:
         - prefix: /
        routeAction:
          single:
            upstream:
              name: default-petclinic-80
              namespace: gloo-system
```

In **apps/staging/** dir we have a Kustomize patch with the staging specific values:

```yaml
apiVersion: enterprise.gloo.solo.io/v1
kind: AuthConfig
metadata:
  name: oauth
  namespace: gloo-system
spec:
  configs:
  - oauth2:
      oidcAuthorizationCode:
        appUrl: https://192.168.64.51
        callbackPath: /callback
        clientId: 2504c599-fcab-4174-9164-a77d3fb5db14 
        clientSecretRef:
          name: oauth
          namespace: gloo-system
        issuerUrl: "http://192.168.64.50:8080/auth/realms/master/"
        scopes:
        - email
        headers:
          idTokenHeader: jwt
```

Note: For each environment or cluster you may change the OIDC configuration.

Infrastructure:

```
./infrastructure/
│   ├── glooedge
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── release.yaml
│   ├── keycloak
│   │   ├── deployment.yaml
│   │   ├── keycloak.yaml
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── service.yaml
│   ├── kustomization.yaml
│   └── sources
│       ├── glooedge.yaml
│       └── kustomization.yaml
```

In **infrastructure/sources/** dir we have the Gloo Edge Enterprise Helm repositories definitions:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: glooedge-e 
spec:
  interval: 30m
  url: https://storage.googleapis.com/gloo-ee-helm
```

Note that with ` interval: 30m` we configure Flux to pull the Helm repository index every 30 minutes.

## Bootstrap staging environment (cluster)

The clusters dir contains the Flux configuration:

```
./clusters/
└── staging
    ├── apps.yaml
    └── infrastructure.yaml
```

In **clusters/staging/** dir we have the Kustomization definitions:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  validation: client
  timeout: 2m
---
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure
  prune: true
  validation: client
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: gloo
      namespace: gloo-system
  timeout: 10m
```

Note that with `path: ./apps/staging` we configure Flux to sync the staging Kustomize overlay and 
with `dependsOn` we tell Flux to create the infrastructure items before deploying the apps.

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Update infrastructure/glooedge/release.yaml license_key with your trial license:

```yaml
  install:
    remediation:
      retries: 5 
  values:
    license_key: "eyJl.........."
    gloo:
      gloo:
        deployment:
          image:
            tag: 1.8.6
```

Set the kubectl context to your staging cluster (in this case it's a local microk8s) and bootstrap Flux:

```sh
flux bootstrap github \
    --context=microk8s \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --token-auth \
    --branch=main \
    --personal \
    --path=clusters/staging
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Wait for the Gloo Edge, Keycloak and Petclinic application installations to complete on staging:

```console
kubectl get pods -A
```

Verify Gloo Edge proxy service can be accessed:

```console
$ kubectl -n gloo-system get service

```


```sh
git add -A && git commit -m "staging update" && git push
```

