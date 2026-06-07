# TD noté — Déploiement Kubernetes

## Prérequis

```powershell
# Vérification des outils
docker --version
k3d version
kubectl version --client
```

## Création du cluster k3d

```powershell
k3d cluster create mon-cluster --agents 2 --port "8080:80@loadbalancer" --k3s-arg "--disable=traefik@server:0"
```

```powershell
# Mise à jour du contexte kubectl
kubectl config use-context k3d-mon-cluster
kubectl config set-cluster k3d-mon-cluster --server=https://127.0.0.1:55694

# Vérification des noeuds
kubectl get nodes
```

---

## Étape 1 — Secret (création impérative)

```powershell
kubectl create secret generic backend-secret --from-literal=admin-token=s3cr3t-token-td
```

```powershell
# Vérification
kubectl get secrets
```

---

## Étape 2 — ConfigMap

```powershell
kubectl apply -f manifests/02-configmap.yaml
```

```powershell
# Vérification
kubectl get configmap frontend-config
kubectl describe configmap frontend-config
```

---

## Étapes 3 & 4 — Deployments (backend + frontend)

Application des manifests dans le bon ordre (dépendances Secret et DNS) :

```powershell
kubectl apply -f manifests/05-service-back.yaml
kubectl apply -f manifests/03-deployment-back.yaml
kubectl apply -f manifests/06-service-front.yaml
kubectl apply -f manifests/04-deployment-front.yaml
```

```powershell
# Vérification globale
kubectl get all

# Étape 3 - 1 pod backend Running READY 1/1
kubectl get pods -l app=backend

# Étape 4 - 2 pods frontend Running READY 1/1
kubectl get pods -l app=frontend

# Étape 4 - variables d'env injectées depuis la ConfigMap
kubectl exec deployment/frontend -- env | findstr BACKEND

# Étape 4 - stratégie rolling update
kubectl get deployment frontend -o jsonpath="{.spec.strategy}"
```

---

## Étape 5 — Application fonctionnelle

Accès à l'application : http://localhost:8080

Screenshot : `screenshots/app-v1.png`

---

## Étape 6 — Rolling update

```powershell
# Dans un terminal séparé, surveiller les pods en temps réel
kubectl get pods -w
```

Modification du fichier `manifests/04-deployment-front.yaml` :
- Ligne modifiée : `image: stephanparichon/epsi-k8s-front:2.0`

```powershell
kubectl apply -f manifests/04-deployment-front.yaml
kubectl rollout status deployment/frontend
```

Screenshot avant : `screenshots/app-v1.png`
Screenshot après : `screenshots/app-v2.png`

---

## Étape 7 — Test du Secret avec curl

```powershell
# Cas 1 : sans token -> 401
curl.exe -X POST http://localhost:8080/api/admin/clear

# Cas 2 : avec le bon token -> 200
curl.exe -X POST http://localhost:8080/api/admin/clear -H "X-Admin-Token: s3cr3t-token-td"
```

---

## État global des ressources déployées

```powershell
kubectl get all
```

Screenshot : `screenshots/kubectl-get-all.png`

---

## Q1 — Que se passe-t-il si vous supprimez manuellement un pod frontend ? Pourquoi ?

Kubernetes recrée automatiquement le pod supprimé. Le Deployment définit un état désiré de 2 replicas : dès qu'un pod disparaît, le ReplicaSet détecte l'écart entre l'état réel (1 pod) et l'état désiré (2 pods) et en recrée un nouveau immédiatement. C'est le mécanisme d'auto-healing de Kubernetes. Le service frontend continue de router le trafic vers le pod restant pendant la recréation, sans interruption visible pour l'utilisateur.
