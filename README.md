# 🚀 Mini-Projet Kubernetes — Déploiement de PayMyBuddy

Déploiement de l'application SpringBoot **PayMyBuddy** avec une base **MySQL** sur Kubernetes, à l'aide de manifests YAML (sans Helm).

---

## 📁 Structure du projet

```
paymybuddy-k8s/
├── Dockerfile                    # Dockerfile officiel PayMyBuddy (amazoncorretto:17-alpine)
├── 01-mysql-secret.yaml          # Secret : credentials MySQL (root / password / db_paymybuddy)
├── 02-mysql-configmap.yaml       # ConfigMap : scripts SQL create.sql + data.sql
├── 03-mysql-storage.yaml         # PersistentVolume (hostPath:/data) + PVC
├── 04-mysql-deployment.yaml      # Deployment MySQL (1 réplicat) + Service ClusterIP
├── 05-paymybuddy-deployment.yaml # Deployment PayMyBuddy + Service NodePort
└── README.md
```

---

## 🏗️ Architecture

```
[Utilisateur]
      │  http://<NODE_IP>:30080
      ▼
┌──────────────────────────┐
│   Service NodePort       │  paymybuddy → port 30080
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│   Deployment PayMyBuddy  │  image: paymybuddy:latest (buildée localement)
│   (SpringBoot :8080)     │  SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/db_paymybuddy
└──────────┬───────────────┘
           │  (DNS interne K8s)
           ▼
┌──────────────────────────┐
│   Service ClusterIP      │  mysql:3306
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│   Deployment MySQL 8.0   │  1 réplicat — init via create.sql + data.sql
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│   PVC → PV hostPath      │  /data/mysql sur le nœud
└──────────────────────────┘
```

---

## ⚙️ Variables d'environnement (issues du Dockerfile officiel)

| Variable | Valeur | Source dans K8s |
|---|---|---|
| `SPRING_DATASOURCE_USERNAME` | `root` | Secret `mysql-secret` |
| `SPRING_DATASOURCE_PASSWORD` | `password` | Secret `mysql-secret` |
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://mysql:3306/db_paymybuddy?...` | Deployment (valeur fixe) |

> L'URL remplace `172.17.0.1` (IP docker0 locale) par `mysql` (nom DNS du Service ClusterIP K8s).
> Le nom de la base `db_paymybuddy` est identique à la configuration locale.

---

## 🔨 Étape 1 — Builder l'image PayMyBuddy

Le Dockerfile copie le JAR précompilé. Il faut donc d'abord compiler avec Maven.

```bash
# 1. Compiler l'application (produit target/paymybuddy.jar)
mvn clean install

# 2a. Avec Minikube — pointer Docker vers le daemon interne
eval $(minikube docker-env)
docker build -t paymybuddy:latest .

# 2b. Avec un cluster kubeadm — builder sur le nœud ou pousser sur un registry
docker build -t paymybuddy:latest .
# Optionnel : pousser sur Docker Hub
# docker tag paymybuddy:latest <user>/paymybuddy:latest && docker push <user>/paymybuddy:latest
```

---

## 🗂️ Étape 2 — Préparer le stockage sur le nœud

```bash
# Sur le nœud worker (ou minikube ssh)
sudo mkdir -p /data/mysql
sudo chmod 777 /data/mysql
```

---

## 🚀 Étape 3 — Déployer sur Kubernetes

```bash
# Appliquer tous les manifests dans l'ordre
kubectl apply -f 01-mysql-secret.yaml
kubectl apply -f 02-mysql-configmap.yaml
kubectl apply -f 03-mysql-storage.yaml
kubectl apply -f 04-mysql-deployment.yaml
kubectl apply -f 05-paymybuddy-deployment.yaml
```

Ou tout d'un coup :

```bash
kubectl apply -f .
```

---

## ✅ Étape 4 — Vérifier

```bash
# État des pods (attendre que les deux soient Running)
kubectl get pods -l app=paymybuddy -w

# Services
kubectl get svc -l app=paymybuddy

# Volumes
kubectl get pv,pvc

# Logs MySQL (vérifier l'init SQL)
kubectl logs -l app=paymybuddy,tier=database --tail=50

# Logs PayMyBuddy
kubectl logs -l app=paymybuddy,tier=frontend --tail=50
```

---

## 🌐 Étape 5 — Accéder à l'application

```bash
# Récupérer l'IP du nœud
kubectl get nodes -o wide

# Ouvrir dans le navigateur :
# http://<NODE_IP>:30080

# Avec Minikube :
minikube service paymybuddy --url
```

**Comptes de test disponibles (depuis data.sql) :**

| Email | Mot de passe |
|---|---|
| `security@mail.com` | `password` |
| `hayley@mymail.com` | `password` |
| `clara@mail.com` | `password` |

---

## 🧹 Suppression complète

```bash
kubectl delete -f .
kubectl delete pv mysql-pv
sudo rm -rf /data/mysql
```

---

## ✅ Bonnes pratiques respectées

- ✅ **Dockerfile officiel** du projet utilisé tel quel (`amazoncorretto:17-alpine`)
- ✅ **3 variables d'env exactes** du Dockerfile : `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `SPRING_DATASOURCE_URL`
- ✅ **Secrets Kubernetes** pour les credentials (pas de mots de passe en clair dans les Deployments)
- ✅ **ConfigMap** avec `create.sql` + `data.sql` montés dans `/docker-entrypoint-initdb.d/` pour l'init automatique de la BDD
- ✅ **PersistentVolume hostPath `/data`** pour stocker les données MySQL sur le nœud (exigence du TP)
- ✅ **Service ClusterIP** pour MySQL (non exposé à l'extérieur)
- ✅ **Service NodePort** pour PayMyBuddy (accès externe port 30080)
- ✅ **initContainer** : PayMyBuddy attend que MySQL soit prêt avant de démarrer
- ✅ **livenessProbe + readinessProbe** sur les deux Deployments
- ✅ **resources requests/limits** définis sur tous les containers
- ✅ **Stratégie Recreate** pour MySQL (single-instance stateful)
- ✅ **imagePullPolicy: IfNotPresent** pour l'image buildée localement
