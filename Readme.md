<h1 align="center">
  <img src="https://flynnt.io/assets/logo-383ae9cd.svg" alt="flynnt" width="100">
+
  <img src="https://www.vectorlogo.zone/logos/argoprojio/argoprojio-ar21.svg" alt="argocd" width="100">
</h1>

<h4 align="center">Deploy ArgoCD on a Flynnt Cluster</h4>

---

This repository contains resources that deploy an instance of [ArgoCD](https://argoproj.github.io/cd/) on a [flynnt](https://flynnt.io) kubernetes cluster.
It is build for private Cloud and on-premise environments.

The deployment is GDPR/DSGVO-compliant. This means, no data stored in your ArgoCD instance will leave your server. As long as you use trusted infrastructure providers (for example your own datacenter) your data is safe.

It is meant to be used for reference and as a blueprint for your own deployment.

#### Special Features of this deployment
- ArgoCD manages itself. After the initial deployment its GitOps all the way.
- ArgoCD configuration uses SSO. You can choose your own provider.
- Secret Management within ArgoCD is done with ksops and age. For more information see below.

> **Note**
>
> Even though this sample is built to be deployed on a flynnt managed kubernetes cluster, you can easily customize this to use a different managed k8s provider.
> In general, almost every technology choice made here is opinionated and exchangable with different products.

Further Resources:

- [Flynnt](https://flynnt.io)
- [Flynnt Documentation](https://docs.flynnt.io)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [ArgoCD with KSops](https://github.com/viaduct-ai/kustomize-sops)


## Used Tooling

We use several open-source tools and stitch them together for a nice, standalone deployment experience.
- Sops and Ksops with Age for secret management
- nginx-ingress and cert-manager for Ingress and public let's encrypt certificates

### Binaries
- [kustomize](https://github.com/kubernetes-sigs/kustomize/releases)
- [sops](https://github.com/getsops/sops/releases)
- [ksops](https://github.com/viaduct-ai/kustomize-sops/releases)
- [age-keygen / age](https://github.com/FiloSottile/age/releases)

## Prerequisites and External Dependencies

- A way to point your preferred DNS record to your nodes for ingress.
- A way to store the initial age secret and share it securely with your team. As an example, a keepass database or a password manager.
- An empty kubernetes cluster and access via kubectl to it. This example uses a [Flynnt Cluster](https://app.flynnt.io). Save the `kubeconfig.yaml` in the root of this repo.
- If you want to make ArgoCD self-managed, it needs access to this repository. There is an example repository secret in `argocd/base/github-repo-secret.yaml`.

## Getting Started

### Generate Age Keys for Encryption

First, we need to generate a secret that we will use to deploy sensitive information into our cluster. Also, we will tell ArgoCD about this secret so ArgoCD itself can use it to decrypt secrets in deployments.

```bash
./age-keygen
```
This generates a public and private keypair. Put them to keepass or a similar secure store.

Next, we will fill out some secrets and encrypt them, so we can commit them to the repository and also use them for ArgoCD.

### Create KSops Secret 
The first secret we need to change is in `argocd/base/age-key-secret.sample.yaml`. This is the key that ArgoCD will use to decrypt ArgoCD deployments/secrets.

Put the generated private key from above into the file and encrypt it like so: 
```bash
./sops --age <age-public-key> --encrypt --encrypted-regex '^(data|stringData)$' argocd/base/age-key-secret.sample.yaml > argocd/base/age-key-secret.enc.yaml
```

### Configure Ingress Domain / DNS

Your DNS record should already point to your k8s nodes. Put your DNS Name in `argocd/base/ingress.yaml` and `argocd/overlays/argocd-cm.yaml`.

### Configure OIDC (optional)
For OIDC, there is a sample configuration in the `ArgoCD Configmap` in `argocd/overlays/argocd-cm.yaml`. The sample uses Google as the Authentication Provider. 
Customize for your own provider. Further documentation can be found in the official ArgoCD documentation, found [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#oidc-configuration-with-dex)

In any case, you will need some kind of OIDC secret that ArgoCD uses.
Create and encrypt the secret
```bash
./sops --age <age-public-key> --encrypt --encrypted-regex '^(data|stringData)$' argocd/base/oidc-secret.sample.yaml > argocd/base/oidc-secret.enc.yaml
```

If you don't want or need this, you can exclude the `oidc-secret.enc.yaml` from Kustomize in `argocd/secret-generator.yaml`.

### Configure git repository access for ArgoCD (optional)

Most likely your repositories are not public and thus not accessible without authentication. In `argocd/base/github-repo-secret.sample.yaml` you find a sample on how to make ArgoCD use authentication when accessing repositories in a Github Organization.
As above, encrypt the secret like this
```bash
./sops --age <age-public-key> --encrypt --encrypted-regex '^(data|stringData)$' argocd/base/github-repo-secret.sample.yaml > argocd/base/github-repo-secret.enc.yaml
```
If you don't want or need this, you can exclude the `github-repo-secret.enc.yaml` from Kustomize in `argocd/secret-generator.yaml`.
Note, that the self-management functionality won't work if ArgoCD has no access to this repository.

### Initial deployment of ArgoCD
Check your connection to the k8s cluster and see if all nodes are fine. We expect the `kubeconfig.yaml` to be in the root of this repo. Download it from the [flynnt dashboard](https://app.flynnt.io) if you haven't already.

```bash
export KUBECONFIG=kubeconfig.yaml
export SOPS_AGE_KEY=<put-private-secret-key-here>
kubectl create namespace argocd
./kustomize build --enable-alpha-plugins --enable-exec ./argocd | kubectl apply -f -
```

#### Make ArgoCD self-managed and expose the UI

We will now switch to a self-managed ArgoCD setup. This means, that you can change the configuration of ArgoCD and the other components by just commiting your changes to this repository and ArgoCD will automatically pick up the changes. GitOps style.
Also, we will deploy Ingress and Cert-Manager resources through ArgoCD. Check `applications.yaml` to see what you are deploying through ArgoCD.

```bash
export KUBECONFIG=kubeconfig.yaml
kubectl apply -f applications.yaml
```

## Encryption of secrets
We use [Mozilla sops](https://github.com/mozilla/sops) and [KSOPS](https://github.com/viaduct-ai/kustomize-sops) with [age](https://github.com/FiloSottile/age).


### Generating keys
```bash
./age-keygen
```
Put public and private key to keepass or a similar secure store.

### Encrypt file
```bash
./sops --age <age-public-key> --encrypt --encrypted-regex '^(data|stringData)$' argocd/base/age-key-secret.yaml > argocd/base/age-key-secret.enc.yaml
```

### Decrypt file
```bash
export SOPS_AGE_KEY=<put-secret-key-here>
./sops --decrypt argocd/base/argocd-cm.enc.yaml > argocd/base/argocd-cm.yaml
```

### Editing the secrets file inline:
```bash
export SOPS_AGE_KEY=<put-secret-key-here>
./sops -i argocd/argocd-cm.enc.yaml
```