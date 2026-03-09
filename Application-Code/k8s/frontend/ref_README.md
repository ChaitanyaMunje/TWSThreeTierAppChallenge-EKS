# Deploying a 3-Tier Application on AWS EKS with ALB Ingress

This guide explains the **complete step-by-step process to deploy a 3-tier application (Frontend + Backend + MongoDB) on AWS EKS** using **AWS Load Balancer Controller, ALB Ingress, and a custom domain**.

The steps include:

* Creating IAM policy for ALB controller
* Configuring OIDC provider
* Installing AWS Load Balancer Controller
* Deploying Kubernetes workloads
* Creating Ingress
* Connecting a custom domain from Hostinger

---

# Architecture

```
Internet
   ↓
Domain (challange.techbfl.in)
   ↓
DNS (Hostinger)
   ↓
AWS ALB (created automatically)
   ↓
Kubernetes Ingress
   ↓
Frontend Service (React)
Backend Service (Node)
   ↓
Pods
   ↓
MongoDB
```

---

# Prerequisites

You must have:

* AWS Account
* EKS Cluster running
* kubectl configured
* eksctl installed
* helm installed
* Docker images available in **public ECR**

Example images used:

```
public.ecr.aws/x6m6m8r8/3-tier-backend:latest
public.ecr.aws/x6m6m8r8/3-tier-frontend:latest
```

---

# Step 1 — Associate IAM OIDC Provider

AWS Load Balancer Controller requires **OIDC provider**.

Run:

```
eksctl utils associate-iam-oidc-provider \
--cluster <cluster-name> \
--approve
```

Example:

```
eksctl utils associate-iam-oidc-provider \
--cluster three-tier-cluster \
--approve
```

---

# Step 2 — Create IAM Policy for Load Balancer Controller

Download policy:

```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create IAM policy:

```
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json
```

Get policy ARN:

```
aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn'
```

Example output:

```
arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy
```

---

# Step 3 — Create IAM Service Account

This creates:

* IAM Role
* Kubernetes Service Account
* Policy Attachment

```
eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn=<policy-arn> \
--approve
```

Example:

```
eksctl create iamserviceaccount \
--cluster=three-tier-cluster \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn=arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```

---

# Step 4 — Install AWS Load Balancer Controller

Add Helm repo:

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install controller:

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=<cluster-name> \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=ap-south-1 \
--set vpcId=<vpc-id>
```

Verify installation:

```
kubectl get pods -n kube-system
```

You should see:

```
aws-load-balancer-controller-xxxxx   Running
```

---

# Step 5 — Deploy MongoDB

Example service:

```
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
  namespace: three-tier
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
```

---

# Step 6 — Deploy Backend

backend-deployment.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: three-tier
spec:
  replicas: 1
  selector:
    matchLabels:
      role: api
  template:
    metadata:
      labels:
        role: api
    spec:
      containers:
      - name: api
        image: public.ecr.aws/x6m6m8r8/3-tier-backend:latest
        ports:
        - containerPort: 3500
        env:
        - name: MONGO_CONN_STR
          value: mongodb://mongodb-svc:27017/todo
```

Service:

```
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: three-tier
spec:
  selector:
    role: api
  ports:
  - port: 3500
    targetPort: 3500
```

---

# Step 7 — Deploy Frontend

frontend-deployment.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
spec:
  replicas: 1
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: frontend
        image: public.ecr.aws/x6m6m8r8/3-tier-frontend:latest
        ports:
        - containerPort: 3000
```

Service:

```
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: three-tier
spec:
  selector:
    role: frontend
  ports:
  - port: 3000
    targetPort: 3000
```

---

# Step 8 — Create Ingress

ingress.yml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mainlb
  namespace: three-tier
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - host: challange.techbfl.in
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 3500

      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 3000
```

Apply ingress:

```
kubectl apply -f ingress.yml
```

Check ingress:

```
kubectl get ingress -n three-tier
```

Example output:

```
mainlb  challange.techbfl.in  k8s-threetie-mainlb-xxxxx.ap-south-1.elb.amazonaws.com
```

---

# Step 9 — Configure DNS (Hostinger)

Login to **Hostinger hPanel**

Navigate:

```
Domains → DNS Zone Editor
```

Add record:

| Type  | Name      | Value                                                  |
| ----- | --------- | ------------------------------------------------------ |
| CNAME | challange | k8s-threetie-mainlb-xxxxx.ap-south-1.elb.amazonaws.com |

Save the record.

DNS propagation usually takes **5–10 minutes**.

---

# Step 10 — Access Application

Open in browser:

```
http://challange.techbfl.in
```

---

# Debugging Commands

Check pods:

```
kubectl get pods -n three-tier
```

Check services:

```
kubectl get svc -n three-tier
```

Check ingress:

```
kubectl describe ingress mainlb -n three-tier
```

Check controller logs:

```
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

# Common Issues

### 1. ADDRESS empty in ingress

Cause:

```
ALB controller not installed
```

Fix:

```
Install AWS Load Balancer Controller
```

---

### 2. Target group unhealthy

Cause:

```
Pod not running
Wrong health check path
```

Fix:

```
Check pod logs
kubectl get pods
```

---

### 3. DNS not resolving

Cause:

```
CNAME record not created
```

Fix:

```
Add CNAME in Hostinger DNS
```

---

# Final Result

Your application will be accessible via:

```
http://challange.techbfl.in
```

Flow:

```
User
 ↓
Domain (Hostinger DNS)
 ↓
AWS ALB
 ↓
Kubernetes Ingress
 ↓
Frontend / API Services
 ↓
Pods
 ↓
MongoDB
```

---

# Future Improvements

Recommended production improvements:

* Enable **HTTPS using AWS ACM**
* Configure **HTTP → HTTPS redirect**
* Add **WAF protection**
* Use **CI/CD pipeline (GitHub Actions + ECR + EKS)**

---
