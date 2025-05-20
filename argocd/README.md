# Déploiement d'Online Boutique avec ArgoCD

Ce guide explique comment configurer et utiliser ArgoCD pour déployer l'application Online Boutique sur un cluster Kubernetes via une approche GitOps.

## Prérequis

- Un cluster Kubernetes fonctionnel (comme le cluster Kind que nous avons déjà configuré)
- kubectl configuré pour accéder à votre cluster
- Git pour gérer les fichiers de configuration
- Un compte GitHub pour héberger votre dépôt de configuration

## Installation d'ArgoCD

1. Créer un namespace pour ArgoCD :

```bash
kubectl create namespace argocd
```

2. Installer ArgoCD dans le cluster :

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

3. Attendre que tous les pods ArgoCD soient prêts :

```bash
kubectl wait --for=condition=available --timeout=600s deployment/argocd-server -n argocd
kubectl wait --for=condition=available --timeout=600s deployment/argocd-repo-server -n argocd
kubectl wait --for=condition=available --timeout=600s deployment/argocd-dex-server -n argocd
kubectl wait --for=condition=available --timeout=600s deployment/argocd-redis -n argocd
kubectl wait --for=condition=ready --timeout=600s statefulset/argocd-application-controller -n argocd
```

4. Exposer l'interface web d'ArgoCD :

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Pour Kind, utiliser le port-forwarding :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

5. Obtenir le mot de passe admin initial :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

6. Accéder à l'interface web ArgoCD sur http://localhost:8080 avec :
   - Username: admin
   - Password: [le mot de passe récupéré à l'étape précédente]

## Structure du dépôt GitOps

Nous avons organisé notre configuration pour ArgoCD dans le dossier `argocd/` avec la structure suivante :

```
argocd/
├── README.md
├── applications/          # Définitions des applications ArgoCD
│   └── online-boutique.yaml
└── manifests/             # Manifests Kubernetes organisés
    ├── online-boutique/
    │   ├── base/
    │   │   └── kustomization.yaml
    │   └── overlays/
    │       ├── dev/
    │       │   └── kustomization.yaml
    │       └── prod/
    │           └── kustomization.yaml
```

## Déploiement de l'application

1. Créer un dépôt Git pour héberger la configuration :

```bash
# Initialiser un nouveau dépôt Git
git init online-boutique-gitops
cd online-boutique-gitops

# Copier les fichiers de configuration
cp -r /path/to/project/argocd/* .

# Committer et pousser les fichiers
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/YOUR_USERNAME/online-boutique-gitops.git
git push -u origin main
```

2. Déployer l'application via l'interface ArgoCD ou avec kubectl :

```bash
kubectl apply -f argocd/applications/online-boutique.yaml
```

3. Surveiller le déploiement dans l'interface web d'ArgoCD ou avec :

```bash
kubectl get applications -n argocd
```

## Synchronisation et mises à jour

Pour mettre à jour l'application, il suffit de modifier les fichiers dans votre dépôt Git et de pousser les changements. ArgoCD détectera automatiquement les modifications et synchronisera l'état de votre cluster avec l'état souhaité défini dans le dépôt.

Pour forcer une synchronisation manuelle :

```bash
kubectl argocd app sync online-boutique -n argocd
```

## Ressources additionnelles

- [Documentation officielle d'ArgoCD](https://argo-cd.readthedocs.io/)
- [Bonnes pratiques GitOps](https://www.weave.works/technologies/gitops/)
