## **Déploiement de l'application Node-shop avec PostgreSQL sur Kubernetes en utilisant Argo CD**

#### Contexte et Objectif

L'objectif de ce projet est de démontrer les compétences DevOps acquises dans le cadre de la certification RNCP 36061. Nous allons déployer une application Node.js nommée `node-shop` avec une base de données PostgreSQL sur un cluster Kubernetes. Nous utiliserons Helm pour la gestion des configurations et Argo CD pour la gestion des déploiements continus. L'application sera déployée dans un namespace dédié `shop-node` et sera auto-scalée à l'aide d'un Horizontal Pod Autoscaler (HPA).

#### Plan du Projet

1. **Présentation de l'Architecture**
2. **Création des Manifests Kubernetes**
3. **Configuration du Chart Helm**
4. **Intégration et Déploiement avec Argo CD**
5. **Mise en Place du Horizontal Pod Autoscaler (HPA)**
6. **Démonstration et Vérifications**

### 1\. Présentation de l'Architecture

L'architecture de notre application se compose des éléments suivants :

- **Application node-shop** : une application e-commerce déployée sur Kubernetes.
- **Base de données PostgreSQL** : utilisée pour stocker les données de l'application.
- **Helm** : outil de gestion de paquets Kubernetes pour automatiser le déploiement.
- **Argo CD** : outil de déploiement continu pour Kubernetes.
- **Horizontal Pod Autoscaler (HPA)** : pour auto-scaling des pods en fonction de l'utilisation du CPU.

### 2\. Création des Manifests Kubernetes

Nous allons commencer par créer les manifests nécessaires pour déployer PostgreSQL et notre application sur Kubernetes.

#### a. Persistent Volume Claim (PVC) pour PostgreSQL

*pvc.yaml*

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: shop-node
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

#### b. Deployment pour PostgreSQL

*deployment-postgres.yaml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: shop-node
  labels:
    app: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - image: postgres:16
        name: postgres
        env:
        - name: POSTGRES_DB
          value: "evershopdb"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-storage
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

#### c. Service pour PostgreSQL

*service-postgres.yaml*

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: shop-node
spec:
  ports:
  - port: 5432
  selector:
    app: postgres
```

### 3\. Configuration du Chart Helm

Nous allons utiliser Helm pour simplifier la gestion et le déploiement de nos manifests Kubernetes.

#### a. Création du Chart Helm

**Structure des fichiers :**

```
charts/
└── node-shop/
    ├── Chart.yaml
    ├── values.yaml
    ├── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── pvc.yaml
        ├── secret.yaml
        ├── deployment-postgres.yaml
        ├── service-postgres.yaml
        ├── hpa.yaml
```

#### b. Fichier Chart.yaml

*charts/node-shop/Chart.yaml*

```yaml
apiVersion: v2
name: node-shop
description: A Helm chart for deploying node-shop and PostgreSQL
type: application
version: 1.0.0
appVersion: "1.0"
```

#### c. Fichier values.yaml

*charts/node-shop/values.yaml*

```yaml
replicaCount: 2

image:
  repository: a2barros78/node-shop
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80

postgres:
  image: postgres:16
  volumeSize: 10Gi
  database: .env.POSTGRES_DB
  user: .env.POSTGRES_USER
  password: .env.POSTGRES_PASSWORD

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths: []
  tls: []
```

#### d. Templates

**i. Deployment pour** `node-shop`

*charts/node-shop/templates/deployment.yaml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-shop
  namespace: shop-node
  labels:
    app: node-shop
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: node-shop
  template:
    metadata:
      labels:
        app: node-shop
    spec:
      containers:
      - name: node-shop
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_HOST
          value: "postgres"
        - name: DATABASE_PORT
          value: "{{ .Values.postgres.port }}"
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-user
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: DATABASE_NAME
          value: "{{ .Values.postgres.database }}"
        resources:
          requests:
            cpu: "100m"
          limits:
            cpu: "500m"
```

**ii. Service pour** `node-shop`

*charts/node-shop/templates/service.yaml*

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-shop
  namespace: shop-node
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: 3000
  selector:
    app: node-shop
```

**iii. Horizontal Pod Autoscaler**

*charts/node-shop/templates/hpa.yaml*

```yaml
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: node-shop-hpa
  namespace: shop-node
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-shop
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.hpa.targetCPUUtilizationPercentage }}
{{- end }}
```

**iv. Secret pour PostgreSQL**

*charts/node-shop/templates/secret.yaml*

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: shop-node
type: Opaque
data:
  postgres-user: {{ .Values.postgres.user | b64enc }}
  postgres-password: {{ .Values.postgres.password | b64enc }}
```

### 4\. Intégration et Déploiement avec Argo CD

#### a. Manifests Argo CD

*argocd/applications.yaml*

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: node-shop
  namespace: shop-node
spec:
  project: default
  source:
    repoURL: 'https://github.com/devopswave/argo-node-shop.git'
    targetRevision: main
    path: charts/node-shop
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: shop-node
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 5\. Mise en Place du Horizontal Pod Autoscaler (HPA)

Le HPA est configuré pour adapter automatiquement le nombre de pods en fonction de l'utilisation du CPU. Il est défini dans le fichier `hpa.yaml` du chart Helm et activé via `values.yaml`.

### 6\. Démonstration et Vérifications

#### a. Pousser les Manifests sur Git

Initialisez votre dépôt Git ou mettez-le à jour et poussez les

changements.

```bash
git init
git add .
git commit -m "Add Helm charts and HPA configuration for node-shop"
git remote add origin https://github.com/devopswave/argo-node-shop.git
git push -u origin main
```

#### b. Synchroniser Argo CD

Argo CD devrait synchroniser automatiquement les changements. Sinon, vous pouvez synchroniser manuellement.

```bash
argocd app sync node-shop
```

#### c. Vérifier le Déploiement

Vérifiez que les pods et les services sont déployés correctement dans le namespace `shop-node` :

```bash
kubectl get pods -n shop-node
kubectl get svc -n shop-node
```

Vérifiez également les logs des pods pour vous assurer qu'il n'y a pas de problèmes :

```bash
kubectl logs deployment/node-shop -n shop-node
kubectl logs deployment/postgres -n shop-node
```

#### d. Vérifier et Tester le HPA

Vérifiez que le HPA est créé et fonctionne correctement :

```bash
kubectl get hpa -n shop-node
```

Vérifiez également les événements associés pour vous assurer que le HPA ajuste correctement le nombre de répliques :

```bash
kubectl describe hpa node-shop-hpa -n shop-node
```
### Points Importants

- **Ressources de CPU** : Assurez-vous que votre déploiement `node-shop` spécifie les demandes et les limites de ressources CPU dans `deployment.yaml`. Le HPA utilise ces valeurs pour le scaling basé sur l'utilisation du CPU.

*Exemple mise à jour `deployment.yaml` pour inclure les ressources :*
```yaml
spec:
  containers:
  - name: node-shop
    resources:
      requests:
        cpu: "100m"
      limits:
        cpu: "500m"
```

- **Installation du Metrics Server** : Si vous utilisez le HPA basé sur l'utilisation du CPU, le Metrics Server doit être installé sur votre cluster Kubernetes pour collecter les métriques nécessaires.

*Installation du Metrics Server :*
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
