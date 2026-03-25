# DevOps App - Plateforme de formation DevOps/DevSecOps/GitOps

Application fil rouge pour pratiquer un workflow complet:

- developpement Node.js
- conteneurisation Docker
- orchestration locale avec Docker Compose
- observabilite (Prometheus, Grafana, Alertmanager)
- securite CI/CD (Trivy, Gitleaks, OPA)
- deploiement declaratif GitOps avec ArgoCD

## Table des matieres

1. [Vue d'ensemble](#vue-densemble)
2. [Structure du repository](#structure-du-repository)
3. [Prerequis](#prerequis)
4. [Application Node.js](#application-nodejs)
5. [Docker et Docker Compose](#docker-et-docker-compose)
6. [Monitoring](#monitoring)
7. [DevSecOps et conformite](#devsecops-et-conformite)
8. [GitOps avec ArgoCD](#gitops-avec-argocd)
9. [CI/CD GitHub Actions](#cicd-github-actions)
10. [Troubleshooting](#troubleshooting)

## Vue d'ensemble

Le projet expose une API HTTP simple avec metriques Prometheus natives:

- `/` : endpoint principal
- `/health` : endpoint de sante (probes, healthchecks)
- `/metrics` : metriques Prometheus (compteurs + histogrammes)

Le dossier contient egalement:

- des manifests Kubernetes et Kustomize
- des exemples Terraform et Ansible
- une stack monitoring complete
- des politiques de securite OPA
- des manifests ArgoCD pour le mode GitOps

## Structure du repository

```text
.
├── app/                         # Application Node.js
├── docker-compose.yml           # Stack applicative locale (app + postgres + redis + nginx)
├── monitoring/                  # Stack Prometheus/Grafana/Alertmanager
├── k8s/                         # Base + overlays Kubernetes
├── gitops/                      # Arborescence GitOps pour ArgoCD
├── infra/                       # Terraform + Ansible
├── policies/                    # Politiques OPA (Dockerfile et Kubernetes)
├── .github/workflows/           # Pipelines CI/CD
├── SECURITY-CHECKLIST.md        # Checklist securite projet
└── .trivyignore                 # Exceptions CVE Trivy (si necessaire)
```

## Prerequis

- Node.js 20+ (18+ compatible selon la matrice CI)
- npm
- Docker + Docker Compose
- kubectl
- (optionnel) argocd CLI
- (optionnel) trivy, gitleaks, conftest

## Application Node.js

### Installation et execution locale

```bash
cd app
npm ci
npm start
```

### Tests

```bash
cd app
npm test
```

### Endpoints

- `http://localhost:3000/`
- `http://localhost:3000/health`
- `http://localhost:3000/metrics`

## Docker et Docker Compose

### Build image applicative

```bash
docker build -t devops-app:1.0.0 app/
```

Le Dockerfile principal (`app/Dockerfile`) est optimise:

- multi-stage
- user non-root
- healthcheck
- image alpine

Un Dockerfile anti-pattern est conserve pour comparaison pedagogique:

- `app/Dockerfile.bad`

### Stack applicative locale

Fichier: `docker-compose.yml`

Services:

- `app`
- `postgres`
- `redis`
- `nginx`

Execution:

```bash
docker compose up -d
docker compose ps
```

Acces:

- `http://localhost` (via nginx)
- `http://localhost:3000/health`

## Monitoring

Le dossier `monitoring/` contient:

- Prometheus (scrape + regles d'alerte)
- Alertmanager (routing webhook)
- Grafana (datasource + dashboard provisionnes)
- node-exporter
- webhook-mock (validation des alertes)

### Lancer la stack monitoring

```bash
cd monitoring
docker compose up -d
docker compose ps
```

Acces:

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3001`
- Alertmanager: `http://localhost:9093`

Identifiants Grafana par defaut (`monitoring/.env`):

- user: `admin`
- password: `devops-training-local`

### Verification rapide

```bash
# Targets Prometheus
curl -s http://localhost:9090/api/v1/targets

# Logs du webhook receiver
cd monitoring
docker compose logs -f webhook-mock
```

## DevSecOps et conformite

### CI security scans

Le workflow CI inclut:

- scan dependances (`trivy fs`)
- scan image Docker (`trivy image`)
- upload SARIF (GitHub Security tab)
- scan IaC (`trivy config`)
- detection de secrets (`gitleaks`)

Fichier:

- `.github/workflows/ci.yml`

### Politiques OPA

- `policies/dockerfile.rego`
- `policies/k8s.rego`

Exemples de regles:

- interdire `root`
- imposer `USER` et `HEALTHCHECK`
- interdire `:latest`
- imposer des `resources.limits` Kubernetes

Validation locale (optionnel):

```bash
conftest test app/Dockerfile --policy policies/
conftest test k8s/base/deployment.yaml --policy policies/
```

### Trivy local (optionnel)

```bash
trivy fs --scanners vuln app/
trivy config infra/
trivy image devops-app:1.0.0
```

Fichiers de support:

- `.trivyignore`
- `SECURITY-CHECKLIST.md`

## GitOps avec ArgoCD

Le dossier `gitops/` contient:

- base Kubernetes de l'application
- overlays `dev`, `staging`, `prod`
- objets ArgoCD: AppProject + 3 Applications

### Installation ArgoCD (cluster local)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Si erreur CRD "annotations too long", utiliser:

```bash
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Mot de passe admin et acces UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

UI: `https://localhost:8443`

### Creer les objets GitOps

```bash
kubectl apply -f gitops/argocd/project.yaml
kubectl apply -f gitops/argocd/app-dev.yaml
kubectl apply -f gitops/argocd/app-staging.yaml
kubectl apply -f gitops/argocd/app-prod.yaml
```

Verification:

```bash
kubectl get applications -n argocd
```

### Note importante registry

ArgoCD synchronise les manifests, mais les pods ne seront Healthy que si l'image existe dans une registry accessible.

Image configuree:

- `ghcr.io/ousmane-sacko/devops-app:<tag>`

Publier l'image et mettre a jour le tag de l'overlay pour un deploiement completement Healthy.

## CI/CD GitHub Actions

Workflow principal: `.github/workflows/ci.yml`

Etapes:

1. Tests Node.js (matrice 18/20/22)
2. Build image Docker (workflow reutilisable)
3. Security scans (Trivy + Gitleaks)
4. Deploy staging (placeholder)
5. Deploy production (placeholder)

## Troubleshooting

### ArgoCD en `Synced` mais pods en erreur

Verifier:

- nom d'image valide (minuscule)
- image disponible dans GHCR
- permissions de pull

```bash
kubectl get pods -n devops-dev
kubectl describe pod <pod-name> -n devops-dev
```

### Prometheus ne scrape pas l'app

Verifier:

- service `app` actif dans la stack monitoring
- endpoint `http://app:3000/metrics` accessible depuis le reseau Docker
- targets dans `http://localhost:9090/targets`

### Trivy indisponible localement

Option Docker:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image devops-app:1.0.0
```

## Licence

Projet de formation interne.
 -Ousmane Sacko 