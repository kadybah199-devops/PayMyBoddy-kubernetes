# 🚀 Mini-Projet Kubernetes — Déploiement de PayMyBuddy

Déploiement de l'application SpringBoot **PayMyBuddy** avec une base **MySQL** sur Kubernetes, à l'aide de manifests YAML 
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
│   image: kady199/paymyboddy:1.0       │
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
kady199/paymybuddy:1.0
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
docker pull kady199/paymybuddy:1.0
```

- Pour Minikube (recommandé si vous utilisez Minikube) :

```bash
minikube image load kady199/paymybuddy:1.0
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
<img width="658" height="470" alt="Capture2" src="https://github.com/user-attachments/assets/e0c7176e-4b68-46ce-8705-466630684dec" />
<img width="803" height="444" alt="9" src="https://github.com/user-attachments/assets/f7f91788-80ec-46ff-99a7-035ed3f6f774" />

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
<img width="803" height="444" alt="9" src="https://github.com/user-attachments/assets/27c80355-ecda-44d0-ad0e-bfc57b9a930f" />
# Vérifier les volumes
kubectl get pv,pvc

# Logs MySQL (vérifier l'initialisation SQL)
kubectl logs -l app=paymybuddy,tier=database --tail=50

# Logs PayMyBuddy (vérifier la connexion à MySQL)
kubectl logs -l app=paymybuddy,tier=frontend --tail=50

# Détail d'un pod en cas de problème
kubectl describe pod -l app=paymybuddy,tier=frontend
```
<img width="658" height="470" alt="Capture2" src="https://github.com/user-attachments/assets/fe8caea0-f64b-4d25-99a0-31e78aee5f3e" />

<img width="977" height="720" alt="8" src="https://github.com/user-attachments/assets/977f1503-1d1a-448c-b0e1-d3b8e228158b" />
---

## 🌐 Étape 4 — Accéder à l'application

```bash
# Récupérer l'IP du nœud
kubectl get nodes -o wide
<img width="1362" height="649" alt="3" src="https://github.com/user-attachments/assets/82ada5b9-46de-434c-9cc5-5fb15581dd51" />

# Ouvrir dans le navigateur
http://<NODE_IP>:30080
```
<img width="1354" height="689" alt="4" src="https://github.com/user-attachments/assets/e6fa3152-75fa-4a4c-b4d6-6b8238ea9a0a" />

**Comptes de test disponibles (depuis data.sql) :**

| Email | Mot de passe |
|---|---|
| `security@mail.com` | `password` |
| `hayley@mymail.com` | `password` |
| `clara@mail.com` | `password` |
| `smith@mail.com` | `password` |
| `lambda@mail.com` | `password` |
<img width="1344" height="607" alt="7" src="https://github.com/user-attachments/assets/4e8d1151-fce1-44b0-bdf9-c6e1a45b142c" />

---

## 🧹 Suppression complète

```bash
kubectl delete -f .
kubectl delete pv mysql-pv
sudo rm -rf /data/mysql

