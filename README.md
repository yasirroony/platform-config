# platform-config

The GitOps config repo: it describes what every environment runs. Argo CD watches it
and makes the cluster match. **Changing a file here is how you deploy.**

| Path | What it is |
| --- | --- |
| `root-app.yaml` | App of apps: apply once, Argo CD manages everything in `apps/` |
| `apps/myapp.yaml` | ApplicationSet that creates `myapp-dev`, `myapp-staging`, `myapp-prod` |
| `apps/platform.yaml` | Applies `platform/` |
| `envs/<env>/myapp/values.yaml` | Helm values and image tag per environment |
| `platform/` | Namespaces (Pod Security `restricted`), limits, quotas, RBAC, network policies |
| `.github/workflows/promote.yml` | Opens a PR that copies an image tag to the next environment |
| `.github/workflows/validate.yml` | Renders the chart for each environment on every PR |
| `CODEOWNERS` | Prod and platform changes need owner approval |

## How a version reaches prod

1. A merge to `main` in **myapp** builds an image and its CI commits the new SHA to `envs/dev/myapp/values.yaml`.
2. Argo CD syncs **dev** automatically.
3. Actions → **promote** (`from: dev`, `to: staging`) opens a PR; merging it deploys **staging**.
4. Actions → **promote** (`from: staging`, `to: prod`) opens a PR; after approval, merge it and press **Sync** on `myapp-prod` in Argo CD.

The image is never rebuilt; only its tag moves. Rollback = `git revert` the promotion commit.

## Bootstrap a cluster (local lab with kind)

```bash
kind create cluster --name cicd-lab
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -n argocd -f root-app.yaml

argocd admin initial-password -n argocd
kubectl -n argocd port-forward svc/argocd-server 8080:443   # https://localhost:8080
```

The `bootstrap` tag in each values file is a placeholder: pods will show `ImagePullBackOff`
until myapp's CI pushes the first real image and bumps dev.

Settings to turn on in GitHub: Actions → General → "Allow GitHub Actions to create and approve pull requests"
(needed by `promote`), and branch protection on `main` requiring a PR and the `validate` check.
