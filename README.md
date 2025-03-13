
# 🚀 Deploy Jenkins in MicroK8s with a Custom Docker Image

This project provides a step-by-step guide to deploy **Jenkins** in **MicroK8s** using Kubernetes manifests and a custom Docker image.  

---

## 📌 Prerequisites  

Ensure you have the following installed on your system:  
- **MicroK8s** (with `kubectl` support)  
- **Docker** (for building and pushing images)  
- **SSH access** to MicroK8s node (if running remotely)  

Enable required MicroK8s add-ons:  
```sh
microk8s enable dns storage ingress registry

```
## 📦 1. Build & Push Custom Jenkins Docker Image 

1.1 Create a Dockerfile
``` sh
# Use official Jenkins LTS image
FROM jenkins/jenkins:lts

# Set root as default user for installation
USER root

# Install additional dependencies
RUN apt-get update && apt-get install -y git curl sudo && rm -rf /var/lib/apt/lists/*

# Switch back to Jenkins user
USER jenkins

# Install necessary Jenkins plugins
RUN jenkins-plugin-cli --plugins workflow-aggregator git blueocean matrix-auth

# Expose ports
EXPOSE 8080 50000

# Start Jenkins
CMD ["jenkins.sh"]
```
1.2 Build & Push Image to MicroK8s Registry
``` sh
docker build -t myjenkins:latest .
docker tag myjenkins:latest localhost:32000/myjenkins:latest
docker push localhost:32000/myjenkins:latest

```
## 🚀 2. Deploy Jenkins in MicroK8s  
**2.1 Create Persistent Volume for Jenkins**  
Create `jenkins-pvc.yaml:`

``` sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
Apply it:  
``` sh
microk8s kubectl apply -f jenkins-pvc.yaml

```
**2.2 Deploy Jenkins with Kubernetes Manifests**
Create `jenkins-deployment.yaml:`
``` sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: localhost:32000/myjenkins:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: jenkins-storage
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-storage
          persistentVolumeClaim:
            claimName: jenkins-pvc

```
Apply it:
``` sh
microk8s kubectl apply -f jenkins-deployment.yaml
```
**2.3 Expose Jenkins Using a Service**

Create `jenkins-service.yaml:`
``` sh
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
spec:
  selector:
    app: jenkins
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```
Apply it:
``` sh
microk8s kubectl apply -f jenkins-service.yaml
```
## 📦 3. Configure Ingress for External Access

**3.1 Create an Ingress Resource**
Create `jenkins-ingress.yaml:`
``` sh
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: jenkins.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins-service
            port:
              number: 8080
```
Apply it:
``` sh
microk8s kubectl apply -f jenkins-ingress.yaml
```
**3.2 Configure Local DNS (If Needed)**

If testing locally, add this to your `/etc/hosts` file:
``` sh
127.0.0.1 jenkins.local
```
## 🚀 4. Access Jenkins

1. Get the Jenkins **Ingress URL**:
``` sh
microk8s kubectl get ingress
```
2. Open Jenkins in a browser:
``` sh
http://jenkins.local
```
3. Retrieve the **Admin Password**:
``` sh
microk8s kubectl logs -f $(microk8s kubectl get pods -l app=jenkins -o jsonpath="{.items[0].metadata.name}")
```
For a public domain, configure DNS records to point to the MicroK8s node’s IP.

✅ **Next Steps**

       🔄 Setup a Jenkins Pipeline

       🌍 Configure SSL using Cert-Manager

       🔐 Secure Jenkins with authentication and backups

 ---
 🙏 **Thank You!**

Thank you for using this guide! If you have any questions or improvements, feel free to contribute or reach out.

✍️ **Author: T.S. Sundar Raj**


