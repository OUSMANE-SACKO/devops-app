# Checklist Securite DevOps

## Images Docker
- [x] Base image minimale (alpine)
- [x] Pas de tag :latest
- [x] User non-root
- [x] HEALTHCHECK present
- [ ] Scan Trivy sans vulnerabilite critique
- [x] .dockerignore complet

## Kubernetes
- [x] Resource limits sur chaque conteneur
- [x] Pas de conteneur privilegie
- [ ] NetworkPolicies en place
- [ ] Secrets via Secret Manager (pas en clair)
- [ ] RBAC configure

## Pipeline
- [ ] Secrets dans GitHub Secrets (pas dans le code)
- [x] Scan de dependances automatique
- [x] Scan d'image automatique
- [x] Detection de secrets (gitleaks)
- [ ] Environnements avec protection

## Infrastructure
- [ ] State Terraform chiffre
- [x] Pas de credentials en dur dans les fichiers IaC
- [ ] Principe du moindre privilege (IAM)
