Understood — you want:

Frontend → Ingress + ELB #1
Backend → Ingress + ELB #2
Then frontend config will call backend ELB URL.

This is possible, but not with one shared NGINX ingress controller and blank host/path because that creates one shared LB and causes conflicts.

You need two separate ingress controllers (two NGINX ingress installations), each with its own LoadBalancer service → two ELBs.

Architecture
Frontend App
    ↓
Frontend Ingress Controller
    ↓
Frontend ELB

Backend App
    ↓
Backend Ingress Controller
    ↓
Backend ELB
Step 1 — Install Frontend NGINX Ingress
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

Frontend ingress controller:

helm install frontend-ingress ingress-nginx/ingress-nginx \
-n ingress-frontend \
--create-namespace \
--set controller.ingressClassResource.name=frontend-nginx \
--set controller.ingressClass=frontend-nginx \
--set controller.service.type=LoadBalancer

This creates:

Frontend ELB
Step 2 — Install Backend NGINX Ingress

Backend ingress controller:

helm install backend-ingress ingress-nginx/ingress-nginx \
-n ingress-backend \
--create-namespace \
--set controller.ingressClassResource.name=backend-nginx \
--set controller.ingressClass=backend-nginx \
--set controller.service.type=LoadBalancer

This creates:

Backend ELB

Now you have 2 separate ELBs.

Step 3 — Check Both ELBs

Run:

kubectl get svc -A

You should see:

ingress-frontend  frontend-ingress-ingress-nginx-controller   LoadBalancer
ingress-backend   backend-ingress-ingress-nginx-controller    LoadBalancer

Each gets separate AWS ELB.

Step 4 — Frontend ingress.yaml

Use frontend ingress class:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
spec:
  ingressClassName: frontend-nginx
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: frontend-service
              port:
                number: 80
Step 5 — Backend ingress.yaml

Use backend ingress class:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
spec:
  ingressClassName: backend-nginx
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: backend-service
              port:
                number: 8080
Get Both ELB URLs

Frontend ELB:

kubectl get svc -n ingress-frontend

Backend ELB:

kubectl get svc -n ingress-backend

Example:

frontend ELB:
a123.us-east-1.elb.amazonaws.com

backend ELB:
b456.us-east-1.elb.amazonaws.com

Use backend ELB inside frontend config.

Important: Two ingresses alone do NOT create two ELBs with the same ingress controller. Two separate ingress controllers + ingressClass are required
