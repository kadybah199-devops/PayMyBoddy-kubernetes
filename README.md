# 🚀 Mini-Projet Kubernetes — Déploiement de PayMyBuddy

Déploiement de l'application SpringBoot **PayMyBuddy** avec une base **MySQL** sur Kubernetes, à l'aide de manifests YAML écrits à la main (sans Helm).

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
┌──────────────────────────────────────────┐
│   Service NodePort                       │
│   paymybuddy → port 30080                │
└──────────────────┬───────────────────────┘
                   ▼
┌──────────────────────────────────────────┐
│   Deployment PayMyBuddy (SpringBoot)     │
│   image: kady199/paymyboddy-backend:latest│
│   SPRING_DATASOURCE_URL=                 │
│   jdbc:mysql://mysql:3306/db_paymybuddy  │
└──────────────────┬───────────────────────┘
                   │  DNS interne Kubernetes
                   ▼
┌──────────────────────────────────────────┐
│   Service ClusterIP  →  mysql:3306       │
└──────────────────┬───────────────────────┘
                   ▼
┌──────────────────────────────────────────┐
│   Deployment MySQL 8.0 (1 réplicat)      │
│   Init auto : create.sql + data.sql      │
└──────────────────┬───────────────────────┘
                   ▼
┌──────────────────────────────────────────┐
│   PersistentVolume  hostPath:/data/mysql │
│   (données stockées sur le nœud)         │
└──────────────────────────────────────────┘
```

---

## 🐳 Image Docker

L'image de l'application est hébergée sur Docker Hub :

```
kady199/paymyboddy-backend:latest
sha256:a69c67a04e69b9021209ea626c47bb42ee2f6980c5519a4e294cf4c16f614ec3
```

Elle a été buildée à partir du Dockerfile officiel du projet :

```dockerfile
FROM amazoncorretto:17-alpine
ARG JAR_FILE=target/paymybuddy.jar
WORKDIR /app
COPY ${JAR_FILE} paymybuddy.jar
ENV SPRING_DATASOURCE_USERNAME=root
ENV SPRING_DATASOURCE_PASSWORD=password
ENV SPRING_DATASOURCE_URL=jdbc:mysql://172.17.0.1:3306/db_paymybuddy
CMD ["java", "-jar", "paymybuddy.jar"]
```

> En Kubernetes, `SPRING_DATASOURCE_URL` est surchargée pour pointer vers
> le Service ClusterIP `mysql:3306` au lieu de l'IP docker0 `172.17.0.1`.

---

## ⚙️ Variables d'environnement

| Variable | Valeur | Source dans K8s |
|---|---|---|
| `SPRING_DATASOURCE_USERNAME` | `root` | Secret `mysql-secret` |
| `SPRING_DATASOURCE_PASSWORD` | `password` | Secret `mysql-secret` |
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://mysql:3306/db_paymybuddy?...` | Deployment (valeur fixe) |

---

## 🗂️ Étape 1 — Préparer le stockage sur le nœud

Les données MySQL sont persistées dans `/data` du nœud, comme demandé dans le TP.

```bash
# Sur le nœud (ou minikube ssh)
sudo mkdir -p /data/mysql
sudo chmod 777 /data/mysql
```

---

## 🚀 Étape 2 — Déployer sur Kubernetes

Avant d'appliquer les manifests vous pouvez soit laisser Kubernetes pull l'image
depuis Docker Hub (le Deployment utilise `imagePullPolicy: Always`), soit
pré-télécharger l'image sur les nœuds.

- Pour récupérer l'image localement (Docker Desktop / Docker CLI disponible) :

```bash
docker pull kady199/paymyboddy-backend:latest
```

- Pour Minikube (recommandé si vous utilisez Minikube) :

```bash
minikube image pull kady199/paymyboddy-backend:latest
```

Ensuite, appliquer les manifests dans l'ordre :

```bash
# Appliquer les manifests dans l'ordre
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

## ✅ Étape 3 — Vérifier le déploiement

```bash
# Suivre le démarrage des pods en temps réel
kubectl get pods -l app=paymybuddy -w

# Vérifier les services
kubectl get svc -l app=paymybuddy

# Vérifier les volumes
kubectl get pv,pvc

# Logs MySQL (vérifier l'initialisation SQL)
kubectl logs -l app=paymybuddy,tier=database --tail=50

# Logs PayMyBuddy (vérifier la connexion à MySQL)
kubectl logs -l app=paymybuddy,tier=frontend --tail=50

# Détail d'un pod en cas de problème
kubectl describe pod -l app=paymybuddy,tier=frontend
```

---

## 🌐 Étape 4 — Accéder à l'application

```bash
# Récupérer l'IP du nœud
kubectl get nodes -o wide

# Ouvrir dans le navigateur
http://<NODE_IP>:30080
```

**Comptes de test disponibles (depuis data.sql) :**

| Email | Mot de passe |
|---|---|
| `security@mail.com` | `password` |
| `hayley@mymail.com` | `password` |
| `clara@mail.com` | `password` |
| `smith@mail.com` | `password` |
| `lambda@mail.com` | `password` |

---

## 🧹 Suppression complète

```bash
kubectl delete -f .
kubectl delete pv mysql-pv
sudo rm -rf /data/mysql
```

---

## ✅ Bonnes pratiques respectées

- ✅ **Image publique Docker Hub** `kady199/paymyboddy-backend:latest` — disponible sur tous les nœuds du cluster sans configuration supplémentaire
- ✅ **imagePullPolicy: Always** — garantit que chaque nœud pull la bonne image, peu importe où le pod est schedulé
- ✅ **3 variables d'env exactes** du Dockerfile officiel : `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `SPRING_DATASOURCE_URL`
- ✅ **Secrets Kubernetes** pour les credentials MySQL (pas de mots de passe en clair dans les Deployments)
- ✅ **ConfigMap** avec `create.sql` + `data.sql` montés dans `/docker-entrypoint-initdb.d/` pour l'initialisation automatique de la base
- ✅ **PersistentVolume hostPath `/data/mysql`** — données MySQL stockées sur le nœud (exigence du TP)
- ✅ **Service ClusterIP** pour MySQL — non exposé à l'extérieur, accessible uniquement via `mysql:3306` en interne
- ✅ **Service NodePort** pour PayMyBuddy — accès externe sur le port `30080`
- ✅ **initContainer** — PayMyBuddy attend que MySQL soit disponible sur le port 3306 avant de démarrer
- ✅ **livenessProbe + readinessProbe** sur les deux Deployments
- ✅ **resources requests/limits** définis sur tous les containers
- ✅ **Stratégie Recreate** pour MySQL — obligatoire pour un pod stateful avec un seul réplicat
- ✅ **Labels cohérents** (`app: paymybuddy`, `tier: database/frontend`) sur toutes les ressources
