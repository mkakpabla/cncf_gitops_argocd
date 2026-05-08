# event-map

Helm chart for the **Event Map** application — stack FastAPI + React/Vite + MongoDB déployée sur Kubernetes via ArgoCD (GitOps).

## Architecture

| Composant | Image | Port | URL |
|-----------|-------|------|-----|
| API (FastAPI/uvicorn) | `ghcr.io/mia/event-map-api` | 9000 | `https://api.<IP>.nip.io` |
| Frontend (React/Vite) | `ghcr.io/mkakpabla/event-map-front` | 4173 | `https://<IP>.nip.io` |
| MongoDB | `mongo:7` | 27017 | interne uniquement |

Tous les objets sont déployés dans le namespace `event-map` (configurable via `namespace`).

## Prérequis

- Kubernetes ≥ 1.25 (testé sur k3s)
- Helm ≥ 3.10
- **Traefik** comme ingress controller (inclus par défaut avec k3s)
- **cert-manager** installé dans le cluster (voir ci-dessous)
- StorageClass `local-path` disponible (incluse par défaut avec k3s)

### Installer cert-manager (une seule fois par cluster)

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true

# Attendre que les pods soient prêts
kubectl -n cert-manager rollout status deployment/cert-manager
kubectl -n cert-manager rollout status deployment/cert-manager-webhook
```


## Configuration

| Paramètre | Description | Défaut |
|-----------|-------------|--------|
| `namespace` | Namespace Kubernetes cible | `event-map` |
| `serverIp` | IP publique du serveur (utilisée pour nip.io) | `-` |
| `certManager.enabled` | Activer la gestion TLS via cert-manager | `true` |
| `certManager.email` | Email pour le compte ACME Let's Encrypt | `mk.akpabla@gmail.com` |
| `certManager.clusterIssuer` | Nom du ClusterIssuer créé | `letsencrypt-prod` |
| `certManager.acmeServer` | URL du serveur ACME | prod Let's Encrypt |
| `mongodb.auth.username` | Utilisateur MongoDB | `user` |
| `mongodb.auth.password` | Mot de passe MongoDB (**à surcharger**) | `changeme` |
| `mongodb.auth.database` | Base de données par défaut | `eventmap` |
| `mongodb.storage.size` | Taille du PVC | `2Gi` |
| `mongodb.storage.storageClassName` | StorageClass | `local-path` |
| `api.image.repository` | Image de l'API | `ghcr.io/mia/event-map-api` |
| `api.image.tag` | Tag de l'image API | `latest` |
| `api.replicaCount` | Nombre de réplicas API | `1` |
| `api.ingress.enabled` | Activer l'ingress API | `true` |
| `front.image.repository` | Image du frontend | `ghcr.io/mkakpabla/event-map-front` |
| `front.image.tag` | Tag de l'image frontend | `latest` |
| `front.replicaCount` | Nombre de réplicas frontend | `1` |
| `front.ingress.enabled` | Activer l'ingress frontend | `true` |

## TLS / HTTPS

Le chart crée automatiquement un `ClusterIssuer` Let's Encrypt et annote les ingresses pour que cert-manager émette les certificats. Les secrets TLS sont stockés dans le namespace `event-map` sous les noms `front-tls` et `api-tls`.

Pour forcer le renouvellement d'un certificat :

```bash
kubectl -n event-map delete secret front-tls api-tls
```

## GitOps avec ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: event-map
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <url-du-repo>
    targetRevision: HEAD
    path: .
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: event-map
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Lint

```bash
helm lint .
```

