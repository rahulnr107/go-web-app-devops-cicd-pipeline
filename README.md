# Go Web App DevOps CI/CD Pipeline

This project demonstrates how a traditional Go web application with no DevOps practices can be transformed into a production-ready CI/CD pipeline using Docker, Kubernetes, Helm, GitHub Actions, ArgoCD and AWS EKS.

The complete workflow automates application build, testing, containerization, image publishing, Kubernetes deployments and continuous delivery using GitOps principles.

---

# Project Workflow

1. Containerize the Go application using Docker.
2. Create an AWS EKS cluster.
3. Deploy the application to Kubernetes using manifests.
4. Configure Service and Ingress resources.
5. Install NGINX Ingress Controller.
6. Package Kubernetes resources using Helm.
7. Implement CI using GitHub Actions.
8. Push Docker images automatically to Docker Hub.
9. Update Helm chart image tags automatically.
10. Deploy automatically using ArgoCD.
11. Validate end-to-end GitOps workflow.

---

# Step 1: Containerize the Application

Build the Docker image:

```bash
docker build -t <dockerhub-username>/go-web-app:v1 .
```

Run the container:

```bash
docker run -p 8080:8080 <dockerhub-username>/go-web-app:v1
```

Verify:

```text
http://localhost:8080/home
```

Push image to Docker Hub:

```bash
docker login
docker push <dockerhub-username>/go-web-app:v1
```

---

# Step 2: Create AWS EKS Cluster

Configure AWS:

```bash
aws configure
```

Create cluster:

```bash
eksctl create cluster \
--name rahnr-cluster \
--region us-east-1
```

Verify:

```bash
kubectl get nodes
```

Expected output:

```text
STATUS = Ready
```

---

# Step 3: Create Kubernetes Manifests

Created the following manifests:

- deployment.yaml
- service.yaml
- ingress.yaml

---

# Step 4: Deploy Application to EKS

Apply all manifests:

```bash
kubectl apply -f k8s/manifests/
```

Verify:

```bash
kubectl get all
```

---

# Step 5: Verify Using NodePort

Temporarily change service type:

```yaml
type: NodePort
```

Verify:

```bash
kubectl get svc
```

Example:

```text
80:30690/TCP
```

Get worker node public IP:

```bash
kubectl get nodes -o wide
```

Access application:

```text
http://<NODE_PUBLIC_IP>:<NODEPORT>/home
```

If application is inaccessible, ensure the NodePort is allowed in the EC2 security group.

---

# Step 6: Install NGINX Ingress Controller

Install controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/aws/deploy.yaml
```

Verify:

```bash
kubectl get pods -n ingress-nginx
```

Expected:

```text
ingress-nginx-controller Running
```

Check LoadBalancer:

```bash
kubectl get svc -n ingress-nginx
```

---

# Step 7: Configure Local DNS Mapping

Resolve LoadBalancer IP:

```bash
nslookup <load-balancer-dns>
```

Edit hosts file:

Linux / WSL:

```bash
sudo vim /etc/hosts
```

Windows:

```text
C:\Windows\System32\drivers\etc\hosts
```

Add:

```text
<LOADBALANCER_IP> go-web-app.local
```

Verify:

```bash
curl http://go-web-app.local/home
```

Expected:

```text
HTML page should be returned successfully.
```

---

# Step 8: Configure Helm

Create chart:

```bash
helm create go-web-app-chart
```

Remove default templates:

```bash
cd go-web-app-chart/templates
rm -rf *
```

Copy manifests:

```bash
cp ../../../k8s/manifests/* .
```

Replace deployment :

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Delete manually created resources:

```bash
kubectl delete deploy go-web-app
kubectl delete svc go-web-app
kubectl delete ingress go-web-app
```

Install Helm chart:

```bash
helm install go-web-app ./go-web-app-chart
```

Verify:

```bash
kubectl get all
kubectl get ingress
```

---

# Step 9: Implement CI using GitHub Actions

The CI pipeline performs:

- Build Application
- Run Unit Tests
- Run GolangCI-Lint
- Build Docker Image
- Push Image to Docker Hub
- Update Helm Chart Image Tag

Required GitHub Secrets:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
TOKEN
```

Push changes:

```bash
git add .
git commit -m "Testing CI/CD"
git push
```

Verify workflow execution from:

```text
GitHub Repository → Actions
```

### GitHub Actions Pipeline

![GitHub Actions Pipeline](screenshots/Screenshot%(202).png)

### Workflow Execution

![Workflow Execution](screenshots/Screenshot%(203).png)

---

# Step 10: Install ArgoCD

Create namespace:

```bash
kubectl create namespace argocd
```

Install ArgoCD:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose ArgoCD UI:

```bash
Change argocd-service to Nodeport
```

Get initial password:

```bash
kubectl get secret argocd-initial-admin-secret \
-n argocd \
-o jsonpath="{.data.password}" | base64 -d
```

Username:

```text
admin
```

---

# Step 11: Configure ArgoCD Application

Create a new application and configure:

Repository URL:

```text
https://github.com/<username>/<repository>
```

Path:

```text
helm/go-web-app-chart
```

Cluster:

```text
https://kubernetes.default.svc
```

Namespace:

```text
default
```

Enable:

- Auto Sync
- Self Heal
- Prune

Create application.

### ArgoCD Application

![ArgoCD Synced Application](screenshots/Screenshot%(204).png)

---

# Step 12: Verify Continuous Deployment

Modify application source code:

```html
<h1>Learn DevOps from Basics</h1>
```

Commit and push:

```bash
git add .
git commit -m "Updated homepage"
git push
```

The following happens automatically:

1. GitHub Actions builds the application.
2. Docker image is created.
3. Image is pushed to Docker Hub.
4. Helm chart image tag is updated.
5. Updated Helm chart is committed.
6. ArgoCD detects the change.
7. New version is deployed automatically.

No manual deployment is required.


# Author

Rahul N R