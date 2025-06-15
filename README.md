

![1](https://github.com/user-attachments/assets/e634eb89-b263-4675-8014-120cc5b99b7e)


# **Kubernetes Resume Challenge: Step-by-Step Guide for Beginners**

Welcome to this comprehensive walkthrough of the **Kubernetes Resume Challenge**! This guide is designed for beginners who want to gain hands-on experience with **Docker, Kubernetes, and DevOps best practices** while building a real-world e-commerce application deployment.  

By the end of this project, you will:  
✅ **Containerize** a web app and database  
✅ **Deploy** to a Kubernetes cluster (AWS/Azure/GCP)  
✅ **Scale, update, and roll back** deployments  
✅ **Use ConfigMaps, Secrets, and Helm**  
✅ **Implement CI/CD and Persistent Storage**  

Let’s get started!  

---

## **Prerequisites**  
Before we begin, ensure you have:  
- A **Docker Hub** account ([Sign Up Here](https://hub.docker.com/))  
- A **cloud account** (AWS, GCP, or Azure) with Kubernetes cluster access  
- **kubectl** installed ([Installation Guide](https://kubernetes.io/docs/tasks/tools/))  
- **Helm** installed (Optional, for advanced steps) ([Installation Guide](https://helm.sh/docs/intro/install/))  
- Basic knowledge of **Linux, Docker, and YAML**  

---

# **Step 1: Containerize the E-Commerce Web App**  

### **A. Create a Dockerfile**  
Navigate to your project directory and create a `Dockerfile`:  

```dockerfile
# Use PHP with Apache
FROM php:7.4-apache

# Install mysqli extension for database connectivity
RUN docker-php-ext-install mysqli

# Copy application files to Apache web directory
COPY . /var/www/html/

# Update database connection to use Kubernetes service (replace placeholders)
ENV DB_HOST=mysql-service
ENV DB_USER=admin
ENV DB_PASSWORD=password
ENV DB_NAME=ecom_db

# Expose port 80 for web traffic
EXPOSE 80
```

### **B. Build and Push the Docker Image**  
1. Build the image:  
   ```sh
   docker build -t yourdockerhubusername/ecom-web:v1 .
   ```
2. Push to Docker Hub:  
   ```sh
   docker push yourdockerhubusername/ecom-web:v1
   ```
**Outcome:** Your web app is now containerized and available on Docker Hub.  

---

# **Step 2: Prepare the Database (MariaDB)**  
Instead of building a custom DB image, we’ll use the official **MariaDB** image in Kubernetes.  

### **A. Create a Database Initialization Script (`db-load-script.sql`)**  
```sql
CREATE DATABASE IF NOT EXISTS ecom_db;
USE ecom_db;

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2)
);

INSERT INTO products (name, price) VALUES ('Laptop', 999.99), ('Phone', 699.99);
```

**Outcome:** This script will initialize the database when the MariaDB pod starts.  

---

# **Step 3: Set Up Kubernetes on a Cloud Provider**  
Choose a cloud provider and create a **managed Kubernetes cluster**:  

| Provider | Service | Guide |
|----------|---------|-------|
| **AWS** | EKS | [EKS Setup](https://aws.amazon.com/eks/) |
| **GCP** | GKE | [GKE Setup](https://cloud.google.com/kubernetes-engine) |
| **Azure** | AKS | [AKS Setup](https://azure.microsoft.com/en-us/products/kubernetes-service/) |

After cluster creation, configure `kubectl`:  
```sh
aws eks --region us-east-1 update-kubeconfig --name my-cluster  # AWS
gcloud container clusters get-credentials my-cluster --region us-central1  # GCP
az aks get-credentials --resource-group my-resource-group --name my-cluster  # Azure
```

**Verify cluster access:**  
```sh
kubectl get nodes
```
**Outcome:** You now have a working Kubernetes cluster!  

---

### Step 3.5: Create Database Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=password=mysecurepassword \
  --from-literal=db_name=ecom_db

# Verify it was created
kubectl get secret db-secret -o yaml
```
---

### **Step 4: Deploy the Website to Kubernetes**  

### **A. Create a `website-deployment.yaml`**  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecom-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ecom-web
  template:
    metadata:
      labels:
        app: ecom-web
    spec:
      containers:
      - name: ecom-web
        image: yourdockerhubusername/ecom-web:v1
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_USER
          value: "admin"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: DB_NAME
          value: "ecom_db"
```

### **B. Deploy MariaDB with Persistent Storage**  
Create `mysql-deployment.yaml`:  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mariadb:10.6
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: MYSQL_DATABASE
          value: "ecom_db"
        - name: MYSQL_USER
          value: "admin"
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

Create `mysql-service.yaml`:  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```

Create a **PersistentVolumeClaim (`mysql-pvc.yaml`)** for database storage:  
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### **C. Apply All Configurations**  
```sh
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f website-deployment.yaml
```

**Verify pods are running:**  
```sh
kubectl get pods
```
**Outcome:** Your website and database are now running in Kubernetes!  

---

# **Step 5: Expose the Website with a Load Balancer**  
Create `website-service.yaml`:  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ecom-web-service
spec:
  type: LoadBalancer
  selector:
    app: ecom-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply it:  
```sh
kubectl apply -f website-service.yaml
```

Get the external IP:  
```sh
kubectl get svc ecom-web-service
```
**Outcome:** Your website is now accessible via the LoadBalancer IP!  

---

# **Step 6: Implement Configuration Management (ConfigMaps)**  
### **A. Add "Dark Mode" Feature Toggle**  
1. Modify your web app to check for `FEATURE_DARK_MODE` environment variable.  
2. Create a **ConfigMap**:  
   ```sh
   kubectl create configmap feature-toggle-config --from-literal=FEATURE_DARK_MODE=true
   ```
3. Update `website-deployment.yaml` to include:  
   ```yaml
   env:
   - name: FEATURE_DARK_MODE
     valueFrom:
       configMapKeyRef:
         name: feature-toggle-config
         key: FEATURE_DARK_MODE
   ```
4. Apply changes:  
   ```sh
   kubectl apply -f website-deployment.yaml
   ```
**Outcome:** Your website now supports dark mode!  

---

# **Step 7: Scale the Application**  
Scale up for increased traffic:  
```sh
kubectl scale deployment ecom-web --replicas=6
```
**Verify scaling:**  
```sh
kubectl get pods
```
**Outcome:** Kubernetes automatically handles increased load!  

---

# **Step 8: Perform a Rolling Update**  
1. Update your app code (e.g., add a promotional banner).  
2. Build & push a new image:  
   ```sh
   docker build -t yourdockerhubusername/ecom-web:v2 .
   docker push yourdockerhubusername/ecom-web:v2
   ```
3. Update `website-deployment.yaml` to use `v2`.  
4. Apply changes:  
   ```sh
   kubectl apply -f website-deployment.yaml
   ```
5. Monitor rollout:  
   ```sh
   kubectl rollout status deployment/ecom-web
   ```
**Outcome:** Zero-downtime update!  

---

# **Step 9: Roll Back a Deployment**  
If the update fails:  
```sh
kubectl rollout undo deployment/ecom-web
```
**Outcome:** The app reverts to the previous stable version!  

---

# **Step 10: Autoscale Based on CPU**  
Create a **Horizontal Pod Autoscaler (HPA)**:  
```sh
kubectl autoscale deployment ecom-web --cpu-percent=50 --min=2 --max=10
```
**Verify autoscaling:**  
```sh
kubectl get hpa
```
**Outcome:** Kubernetes scales pods automatically under load!  

---

# **Step 11: Implement Liveness & Readiness Probes**  
Update `website-deployment.yaml`:  
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```
**Outcome:** Kubernetes ensures only healthy pods serve traffic!  

---

# **Step 12: Document Everything**  
- Push all files to **GitHub**  
- Write a **README.md** explaining each step  
- Consider a **blog post** on Medium/Dev.to  

---

# **Extra Credit (Advanced Steps)**  
### **1. Package Everything with Helm**  
```sh
helm create ecom-app
helm install ecom-app ./ecom-app
```
### **2. Implement CI/CD with GitHub Actions**  
Example workflow (`.github/workflows/deploy.yml`):  
```yaml
name: Deploy to Kubernetes
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Build Docker Image
      run: docker build -t yourdockerhubusername/ecom-web:latest .
    - name: Push to Docker Hub
      run: |
        echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
        docker push yourdockerhubusername/ecom-web:latest
    - name: Deploy to Kubernetes
      run: kubectl apply -f k8s/
```

---

# **Conclusion**  
Congratulations! 🎉 You’ve successfully:  
✔ Deployed a **scalable e-commerce app** on Kubernetes  
✔ Used **ConfigMaps, Secrets, Helm, and CI/CD**  
✔ Gained **real-world DevOps experience**  

Now, **add this project to your resume** and showcase your Kubernetes skills! 🚀  

## **Troubleshooting Summary**

During this project, I encountered and resolved several real-world issues across deployment stages. Below is a detailed breakdown of the challenges and their solutions, along with key takeaways.

### **1️⃣ Docker Image & Database Connection**  
**Issue**: The web app failed to connect to MySQL despite successful deployment.  
**Root Cause**:  
- Incorrect hostname in connection.php (localhost instead of the Kubernetes service name mysql-service)  
- Missing environment variables for database credentials  

**Fix**:  
- Updated DB_HOST to mysql-service in the PHP connection script  
- Rebuilt the Docker image and redeployed:  

```bash
docker build -t username/ecom-web:v2 .
docker push username/ecom-web:v2
kubectl set image deployment/ecom-web ecom-web=username/ecom-web:v2
Verification:

bash
kubectl exec -it [POD_NAME] -- curl http://localhost/ready.php  # Check DB connectivity
2️⃣ LoadBalancer Not Serving Website
Issue: The external IP assigned by the LoadBalancer did not display the website.
Diagnosis Steps:

bash
kubectl logs -l app=ecom-web
kubectl describe svc ecom-web-service | grep Selector
kubectl run busybox --rm -it --image=busybox -- wget -O- http://ecom-web-service
Root Cause:
The app was failing silently due to the unresolved database hostname (see Issue 1).

Fix:

Corrected the database connection issue first

Ensured the LoadBalancer service was properly configured:

yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: ecom-web
3️⃣ Secrets Needed Earlier Than Expected
Original Plan: Implement Secrets in Step 12 (Config Management).
Problem: The app required DB_PASSWORD much earlier (Step 3.5).
Fix:

bash
kubectl create secret generic db-secret --from-literal=password=mysecurepassword
Added to website-deployment.yaml:

yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
4️⃣ ConfigMap – Dark Mode Feature Toggle
Issue: The FEATURE_DARK_MODE ConfigMap had no visible effect.
Root Cause:

Frontend didn't read the environment variable

PHP backend didn't pass the flag to the template

Fix:
Updated index.php:

php
$dark_mode = getenv('FEATURE_DARK_MODE') === 'true';
Verification:

bash
kubectl get configmap feature-toggle-config -o yaml
5️⃣ Probes Failing (CrashLoopBackOff)
Issue: Pods crashed repeatedly after adding livenessProbe and readinessProbe.
Root Cause: The app lacked /health and /ready endpoints.

Fix:
Added health.php:

php
// health.php
http_response_code(200);
echo "OK";
Updated deployment:

yaml
livenessProbe:
  httpGet:
    path: /health.php
    port: 80
  initialDelaySeconds: 15
6️⃣ Config Changes Not Reflecting
Issue: Updates to website-deployment.yaml didn't trigger pod updates.
Fix:

bash
kubectl rollout restart deployment ecom-web
# OR
kubectl apply -f website-deployment.yaml  # With changed image tag
Lessons Learned & Best Practices
✅ Debugging Methodology:

Layer-by-layer checks (network → service → app)

Use temporary pods (busybox, mysql-client) for connectivity tests

✅ Kubernetes Best Practices:

Always use Services for inter-pod communication (DNS names like mysql-service)

Create Secrets & ConfigMaps before deployments that reference them

Probes are critical—ensure /health and /ready endpoints exist

✅ CI/CD Improvements:

Rebuild and retag images for every change (e.g., v1, v2)

Automate rollouts using kubectl rollout status

**Next Steps:**  
- Explore **Ingress Controllers (Nginx, Traefik)**  
- Learn **Kubernetes Monitoring (Prometheus, Grafana)**  
- Dive into **Service Meshes (Istio, Linkerd)**  

Happy Learning! 😊

🙏 Acknowledgements
Original project source https://cloudresumechallenge.dev/docs/extensions/kubernetes-challenge/.
This project was made possible by leveraging the excellent Kubernetes tutorials created by Anton Putz. Special thanks for these key resources:

Tutorial	Concept Learned	Direct Link
Lesson #099	Persistent Volumes	github.com/antonputra/tutorials/tree/main/lessons/099
Lesson #071	Horizontal Pod Autoscaling (HPA)	github.com/antonputra/tutorials/tree/main/lessons/071
Lesson #110	Network Policies	github.com/antonputra/tutorials/tree/main/lessons/110
Lesson #134	PVC Monitoring with Prometheus	github.com/antonputra/tutorials/tree/main/lessons/134
Lesson #122	Slack Alerting	github.com/antonputra/tutorials/tree/main/lessons/122
Additional Resources:

Anton's Full Tutorial Repository https://github.com/antonputra/tutorials

Anton Putz YouTube Channel youtube.com/antonputra
