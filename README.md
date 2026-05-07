# event-map

Helm chart for the **Event Map** application — stack FastAPI + React/Vite + MongoDB déployée sur Kubernetes via ArgoCD (GitOps).

## Architecture

| Composant | Image | Port |
|-----------|-------|------|
| API (FastAPI/uvicorn) | `ghcr.io/mia/event-map-api` | 9000 |
| Frontend (React/Vite) | `ghcr.io/mia/event-map-front` | 5173 |
| MongoDB | `mongo:7` | 27017 |

Tous les objets sont déployés dans le namespace `event-map` (configurable via `namespace`).

## Prérequis

- Kubernetes ≥ 1.25
- Helm ≥ 3.10
- Ingress Controller nginx (pour exposer l'API et le frontend)
- StorageClass disponible pour le PVC MongoDB (ou définir `mongodb.storage.storageClassName`)

## Installation

```bash
# Ajout du repo (si packagé)
helm install event-map . \
  --namespace event-map \
  --create-namespace \
  --set mongodb.auth.password=<mot-de-passe-secret>
```

## Mise à jour

```bash
helm upgrade event-map . \
  --set mongodb.auth.password=<mot-de-passe-secret>
```

## Désinstallation

```bash
helm uninstall event-map --namespace event-map
```

## Configuration

| Paramètre | Description | Défaut |
|-----------|-------------|--------|
| `namespace` | Namespace Kubernetes cible | `event-map` |
| `mongodb.image.tag` | Tag de l'image MongoDB | `7` |
| `mongodb.auth.username` | Utilisateur MongoDB | `user` |
| `mongodb.auth.password` | Mot de passe MongoDB (**à surcharger**) | `changeme` |
| `mongodb.auth.database` | Base de données par défaut | `eventmap` |
| `mongodb.storage.size` | Taille du PVC | `2Gi` |
| `mongodb.storage.storageClassName` | StorageClass (vide = défaut) | `""` |
| `api.image.repository` | Image de l'API | `ghcr.io/mia/event-map-api` |
| `api.image.tag` | Tag de l'image API | `latest` |
| `api.replicaCount` | Nombre de réplicas API | `1` |
| `api.ingress.enabled` | Activer l'ingress API | `true` |
| `api.ingress.host` | Hostname de l'API | `api.event-map.local` |
| `front.image.repository` | Image du frontend | `ghcr.io/mia/event-map-front` |
| `front.image.tag` | Tag de l'image frontend | `latest` |
| `front.replicaCount` | Nombre de réplicas frontend | `1` |
| `front.env.VITE_API_URL` | URL de l'API côté client | `http://api.event-map.local` |
| `front.ingress.enabled` | Activer l'ingress frontend | `true` |
| `front.ingress.host` | Hostname du frontend | `event-map.local` |

## GitOps avec ArgoCD

Pointer l'`Application` ArgoCD vers ce dépôt :

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
