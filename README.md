# 🚀 Mini-Projet Kubernetes — Déploiement de PayMyBuddy

Application SpringBoot connectée à une base MySQL, déployée avec des manifests Kubernetes.

---

## 📁 Structure des manifests

```
paymybuddy-k8s/
├── 01-mysql-secret.yaml         # Secret Kubernetes (credentials MySQL)
├── 02-mysql-storage.yaml        # PersistentVolume + PersistentVolumeClaim
├── 03-mysql-deployment.yaml     # Deployment MySQL + Service ClusterIP
├── 04-paymybuddy-deployment.yaml # Deployment PayMyBuddy + Service NodePort
└── README.md
```

---

## 🏗️ Architecture

```
[Utilisateur]
      │
      │ http://<NODE_IP>:30080
      ▼
┌─────────────────────┐
│  Service NodePort   │  ← paymybuddy:30080
│  (PayMyBuddy)       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Deployment          │
│  PayMyBuddy         │  ← eazytraining/paymybuddy:latest
│  (SpringBoot:8080)  │
└────────┬────────────┘
         │ SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/paymybuddy
         ▼
┌─────────────────────┐
│  Service ClusterIP  │  ← mysql:3306 (interne uniquement)
│  (MySQL)            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Deployment MySQL   │  ← mysql:8.0
│  (1 réplicat)       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  PVC → PV           │  ← hostPath: /data/mysql (nœud)
└─────────────────────┘
```

---

## ⚙️ Étapes de déploiement

### Prérequis
- Cluster Kubernetes opérationnel (Minikube, kubeadm, etc.)
- `kubectl` configuré

### 1. Créer le répertoire de données sur le nœud

```bash
# Sur le nœud worker (ou minikube ssh)
sudo mkdir -p /data/mysql
sudo chmod 777 /data/mysql
```

### 2. Appliquer les manifests dans l'ordre

```bash
# 1. Secrets (credentials)
kubectl apply -f 01-mysql-secret.yaml

# 2. Stockage persistant
kubectl apply -f 02-mysql-storage.yaml

# 3. Déploiement MySQL + Service ClusterIP
kubectl apply -f 03-mysql-deployment.yaml

# 4. Déploiement PayMyBuddy + Service NodePort
kubectl apply -f 04-paymybuddy-deployment.yaml
```

Ou tout d'un coup :

```bash
kubectl apply -f .
```

### 3. Vérifier le déploiement

```bash
# Vérifier les pods
kubectl get pods -l app=paymybuddy

# Vérifier les services
kubectl get svc -l app=paymybuddy

# Vérifier les volumes
kubectl get pv,pvc

# Logs MySQL
kubectl logs -l app=paymybuddy,tier=database

# Logs PayMyBuddy
kubectl logs -l app=paymybuddy,tier=frontend
```

### 4. Accéder à l'application

```bash
# Récupérer l'IP du nœud
kubectl get nodes -o wide

# Accéder à l'application
# http://<NODE_IP>:30080

# Avec Minikube :
minikube service paymybuddy --url
```

---

## 🔧 Variables d'environnement PayMyBuddy

| Variable | Valeur | Description |
|---|---|---|
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://mysql:3306/paymybuddy?...` | URL de connexion MySQL |
| `SPRING_DATASOURCE_USERNAME` | (depuis Secret) | Utilisateur MySQL |
| `SPRING_DATASOURCE_PASSWORD` | (depuis Secret) | Mot de passe MySQL |
| `SPRING_JPA_HIBERNATE_DDL_AUTO` | `update` | Gestion du schéma JPA |

---

## 🧹 Suppression complète

```bash
kubectl delete -f .
kubectl delete pv mysql-pv
# Sur le nœud :
sudo rm -rf /data/mysql
```

---

## ✅ Bonnes pratiques respectées

- ✅ Secrets Kubernetes pour les credentials (pas de mots de passe en clair dans les Deployments)
- ✅ PersistentVolume avec `hostPath: /data` pour la persistance des données MySQL
- ✅ Service `ClusterIP` pour MySQL (non exposé à l'extérieur)
- ✅ Service `NodePort` pour PayMyBuddy (accès externe)
- ✅ `initContainer` pour attendre que MySQL soit prêt avant de démarrer PayMyBuddy
- ✅ `livenessProbe` et `readinessProbe` sur les deux services
- ✅ `resources` (requests/limits) définis sur tous les containers
- ✅ Stratégie `Recreate` pour MySQL (compatible avec 1 seul réplicat stateful)
- ✅ Labels cohérents sur toutes les ressources
