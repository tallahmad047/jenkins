# Guide Officiel : Déploiement de Jenkins sur Kubernetes

Ce document explique les étapes nécessaires pour déployer Jenkins sur un cluster Kubernetes. Nous fournirons un fichier YAML complet ainsi que les commandes associées pour un déploiement réussi.

---

## Prérequis

Avant de commencer, assurez-vous que :
- Un cluster Kubernetes est configuré et fonctionnel.
- `kubectl` est installé et configuré pour interagir avec le cluster.
- Le namespace `lcf-k8s-devops` est créé ou sera créé au cours du déploiement.
- Vous disposez d'une classe de stockage compatible (par ex. `hostpath` ou une autre StorageClass configurée).

---

## Création du Namespace

Si le namespace n'existe pas, créez-le :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lcf-k8s-devops
```

Appliquez cette configuration :
```bash
kubectl apply -f namespace.yaml
```

---

## Fichier YAML de Déploiement

Voici le fichier YAML complet pour déployer Jenkins. Il inclut :
1. Un ServiceAccount avec des permissions RBAC.
2. Un PersistentVolumeClaim pour le stockage des données Jenkins.
3. Une Deployment pour exécuter Jenkins.
4. Deux services : un pour l'accès interne (ClusterIP) et un pour l'accès externe (NodePort).

### Code Complet
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: lcf-k8s-devops
  labels:
    app: jenkins
    criticity: high
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin-clusterrole
  labels:
    app: jenkins
    criticity: high
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin-clusterrolebinding
  labels:
    app: jenkins
    criticity: high
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin-clusterrole
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: lcf-k8s-devops
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-jenkins
  namespace: lcf-k8s-devops
spec:
  storageClassName: "hostpath"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: lcf-k8s-devops
  labels:
    app: jenkins
    criticity: high
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk21
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
              claimName: pvc-jenkins
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service-jnlp
  namespace: lcf-k8s-devops
  labels:
    app: jenkins
    criticity: high
spec:
  selector:
    app: jenkins-server
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      name: jenkins-http-interne
    - port: 50000
      targetPort: 50000
      name: jenkins-jnlp
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service-http
  namespace: lcf-k8s-devops
  labels:
    app: jenkins
    criticity: high
spec:
  selector:
    app: jenkins-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
      name: jenkins-url
```

---

## Commandes pour le Déploiement

1. Appliquez le namespace (si ce n'est pas fait) :
   ```bash
   kubectl apply -f namespace.yaml
   ```

2. Appliquez le fichier YAML :
   ```bash
   kubectl apply -f jenkins-deployment.yaml
   ```

3. Vérifiez les pods :
   ```bash
   kubectl get pods -n lcf-k8s-devops
   ```

4. Accédez à Jenkins via l'URL suivante :
   - Si vous utilisez NodePort : `http://<IP-Node>:32000`

5. Récupérez le mot de passe initial :
   ```bash
   kubectl logs <jenkins-pod-name> -n lcf-k8s-devops
   ```

---

## Améliorations Futures

1. **Sécurité** : Ajoutez un Ingress avec SSL pour un accès sécurisé.
2. **Monitoring** : Configurez Prometheus et Grafana pour superviser Jenkins.
3. **Backups** : Implémentez une stratégie de sauvegarde pour le volume persistant.

---

Avec ces étapes, vous aurez un Jenkins pleinement fonctionnel déployé sur Kubernetes, prêt à gérer vos pipelines CI/CD.

