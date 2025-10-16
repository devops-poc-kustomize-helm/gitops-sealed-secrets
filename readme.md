
# GitOps Sealed Secrets Repository

This repository contains **SealedSecrets** for applications and instructions to deploy them using **ArgoCD**.

---

## 1️⃣ Install Sealed Secrets Controller

The Sealed Secrets controller is required to decrypt SealedSecrets into regular Kubernetes Secrets.

```bash
# Add Bitnami Sealed Secrets Helm repository
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

# Install Sealed Secrets controller in the 'sealed-secrets' namespace
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace sealed-secrets --create-namespace
````

---

## 2️⃣ Get the Sealed Secrets Public Certificate

You need the public certificate to encrypt secrets before committing them to Git:

```bash
kubectl -n sealed-secrets get secret -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o jsonpath="{.items[0].data['tls\.crt']}" | base64 -d > sealed-secrets-cert.pem
```

This certificate (`sealed-secrets-cert.pem`) can be used with `kubeseal` or your custom scripts to generate SealedSecrets.

---

## 3️⃣ Create a SealedSecret

Start with a normal Kubernetes Secret YAML, e.g., `secret-plain.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  namespace: my-app-namespace
type: Opaque
stringData:
  username: myuser
  password: mypassword
```

Encrypt it using the public certificate:

```bash
# Using the certificate file
kubeseal --cert sealed-secrets-cert.pem < secret-plain.yaml > app-credentials-sealed.yaml

# Or using the controller directly
kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format=yaml < secret-plain.yaml > app-credentials.yaml
```

The resulting YAML (`app-credentials-sealed.yaml` or `app-credentials.yaml`) is safe to commit to Git.

---

## 4️⃣ GitOps Deployment via ArgoCD

Create an ArgoCD Application manifest to deploy all SealedSecrets from this repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/devops-poc-kustomize-helm/gitops-sealed-secrets.git'
    targetRevision: main
    path: apps
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: sealed-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

* **`path: apps`** → folder containing your SealedSecrets YAMLs.
* **`namespace: sealed-secrets`** → namespace where ArgoCD applies the SealedSecrets CRDs.
* The Sealed Secrets controller will automatically decrypt them into **Secrets** in the namespaces defined in each SealedSecret `template`.

---

## 5️⃣ Workflow

1. Create or update a plain secret locally.
2. Encrypt it using `kubeseal` or your custom script.
3. Commit the SealedSecret YAML to the repository.
4. ArgoCD automatically syncs and deploys the SealedSecrets.
5. Applications in the corresponding namespace can now use the Secrets.

---

## 6️⃣ Best Practices

* Keep **SealedSecrets in the same namespace as the resulting Secret**.
* Do **not commit plain secrets** to Git.
* Organize secrets by application and environment for clarity, e.g.,

```
apps/
  dev/
    app1.yaml
    app2.yaml
  test/
    app1.yaml
```

* Use **automated ArgoCD sync** for continuous deployment.

---
