# Fluxcd and Gloo Edge

This repo was cloned from the [Fluxcd Project](https://github.com/fluxcd/flux2-kustomize-helm-example)

## Prerequisites

Request a Gloo Edge Enterprise [trial license](https://lp.solo.io/request-trial).

You will need a Kubernetes cluster version 1.16 or newer and kubectl version 1.18.
For a quick local test, you can use [Microk8s](https://www.solo.io/blog/a-local-development-kubernetes-environment-for-gloo-edge/).
Any other Kubernetes should work, in general the cluster needs to support Kubernetes ServiceType Loadbalancer.

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

- **apps** dir contains kubernetes application manifests, which are the customized per cluster.
- **infrastructure** dir contains common infrastructure services such as datastores, ingress controllers, etc. and Helm repository definitions. in theis case [Gloo Edge Enterprise](https://docs.solo.io/gloo-edge/latest/) 
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

Note that with ` interval: 30m` we configure Flux to pull the Helm repository index every five minutes.

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

Watch for the Helm releases being install on staging:

```console
$ watch flux get helmreleases --all-namespaces 
NAMESPACE	NAME   	REVISION	SUSPENDED	READY	MESSAGE                          
nginx    	nginx  	5.6.14  	False    	True 	release reconciliation succeeded	
podinfo  	podinfo	5.0.3   	False    	True 	release reconciliation succeeded	
redis    	redis  	11.3.4  	False    	True 	release reconciliation succeeded
```

Verify that the demo app can be accessed via ingress:

```console
$ kubectl -n nginx port-forward svc/nginx-ingress-controller 8080:80 &

$ curl -H "Host: podinfo.staging" http://localhost:8080
{
  "hostname": "podinfo-59489db7b5-lmwpn",
  "version": "5.0.3"
}
```

Bootstrap Flux on production by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production
```

Watch the production reconciliation:

```console
$ watch flux get kustomizations
NAME          	REVISION                                        READY
apps          	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
flux-system   	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
infrastructure	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
```

## Encrypt Kubernetes secrets

In order to store secrets safely in a Git repository,
you can use Mozilla's SOPS CLI to encrypt Kubernetes secrets with OpenPGP or KMS.

Install [gnupg](https://www.gnupg.org/) and [sops](https://github.com/mozilla/sops):

```sh
brew install gnupg sops
```

Generate a GPG key for Flux without specifying a passphrase and retrieve the GPG key ID:

```console
$ gpg --full-generate-key
Email address: fluxcdbot@users.noreply.github.com

$ gpg --list-secret-keys fluxcdbot@users.noreply.github.com
sec   rsa3072 2020-09-06 [SC]
      1F3D1CED2F865F5E59CA564553241F147E7C5FA4
```

Create a Kubernetes secret on your clusters with the private key:

```sh
gpg --export-secret-keys \
--armor 1F3D1CED2F865F5E59CA564553241F147E7C5FA4 |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin
```

Generate a Kubernetes secret manifest and encrypt the secret's data field with sops:

```sh
kubectl -n redis create secret generic redis-auth \
--from-literal=password=change-me \
--dry-run=client \
-o yaml > infrastructure/redis/redis-auth.yaml

sops --encrypt \
--pgp=1F3D1CED2F865F5E59CA564553241F147E7C5FA4 \
--encrypted-regex '^(data|stringData)$' \
--in-place infrastructure/redis/redis-auth.yaml
```

Add the secret to `infrastructure/redis/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: redis
resources:
  - namespace.yaml
  - release.yaml
  - redis-auth.yaml
```

Enable decryption on your clusters by editing the `infrastructure.yaml` files:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  # content omitted for brevity
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

Export the public key so anyone with access to the repository can encrypt secrets but not decrypt them:

```sh
gpg --export -a fluxcdbot@users.noreply.github.com > public.key
```

Push the changes to the main branch:

```sh
git add -A && git commit -m "add encrypted secret" && git push
```

Verify that the secret has been created in the `redis` namespace on both clusters:

```sh
kubectl --context staging -n redis get secrets
kubectl --context production -n redis get secrets
```

You can use Kubernetes secrets to provide values for your Helm releases:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
spec:
  # content omitted for brevity
  values:
    usePassword: true
  valuesFrom:
  - kind: Secret
    name: redis-auth
    valuesKey: password
    targetPath: password
```

Find out more about Helm releases values overrides in the
[docs](https://toolkit.fluxcd.io/components/helm/helmreleases/#values-overrides).


## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p clusters/dev
```

Copy the sync manifests from staging:

```sh
cp clusters/staging/infrastructure.yaml clusters/dev
cp clusters/staging/apps.yaml clusters/dev
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev/apps.yaml` to `path: ./apps/dev`. 

Push the changes to the main branch:

```sh
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev
```

## Identical environments

If you want to spin up an identical environment, you can bootstrap a cluster
e.g. `production-clone` and reuse the `production` definitions.

Bootstrap the `production-clone` cluster:

```sh
flux bootstrap github \
    --context=production-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production-clone
```

Pull the changes locally:

```sh
git pull origin main
```

Create a `kustomization.yaml` inside the `clusters/production-clone` dir:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../production/infrastructure.yaml
  - ../production/apps.yaml
```

Note that besides the `flux-system` kustomize overlay, we also include
the `infrastructure` and `apps` manifests from the production dir.

Push the changes to the main branch:

```sh
git add -A && git commit -m "add production clone" && git push
```

Tell Flux to deploy the production workloads on the `production-clone` cluster:

```sh
flux reconcile kustomization flux-system \
    --context=production-clone \
    --with-source 
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.

This repository contains the following GitHub CI workflows:

* the [test](./.github/workflows/test.yaml) workflow validates the Kubernetes manifests and Kustomize overlays with kubeval
* the [e2e](./.github/workflows/e2e.yaml) workflow starts a Kubernetes cluster in CI and tests the staging setup by running Flux in Kubernetes Kind
