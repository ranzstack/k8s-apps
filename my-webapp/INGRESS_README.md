# Ingress — How External Traffic Gets Into the Cluster

## The Problem

A Service alone is only reachable **inside** the cluster. Without Ingress, there is no permanent
way for external traffic (browser, API clients) to reach your app. `kubectl port-forward` is a
temporary debugging tunnel — not a real solution for production.

## The Solution: Ingress

Ingress sits at the edge of the cluster and routes external HTTP/HTTPS traffic to the right Service.

```
Internet
    │
    ▼
Ingress Controller  (the software — installed once as cluster infrastructure)
    │
    ▼
Ingress Rules       (your YAML — defines which domain/path goes to which Service)
    │
    ▼
Service             (routes to pods via label selector)
    │
    ▼
Pods                (your actual app)
```

## Two Parts of Ingress

### 1. Ingress Controller (cluster infrastructure — not in this repo)
The actual software that handles incoming traffic. Installed once by the platform/DevOps team
during cluster setup. Developers don't touch this.

| Environment | Ingress Controller |
|-------------|-------------------|
| Local (kind) | nginx ingress controller |
| AWS EKS | AWS Load Balancer Controller |
| GKE | Google Cloud Load Balancer |

In EKS, the AWS Load Balancer Controller automatically provisions an AWS ALB (Application Load
Balancer) for each Ingress resource you create. That ALB gets a real public DNS name.

How it was installed locally (one-time, not in Git):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.2/deploy/static/provider/kind/deploy.yaml
kubectl label node learning-control-plane ingress-ready=true
```

In EKS this is done via Helm (Kubernetes package manager) during cluster provisioning:
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system
```

### 2. Ingress Resource (this repo — managed by ArgoCD)
The routing rules for your specific app. This is what lives in Git and gets deployed by ArgoCD
like any other app config. See `ingress.yaml`.

## Local vs EKS: Domain Names

Locally, we fake the domain by adding it to `/etc/hosts` on your Mac:
```bash
echo "127.0.0.1 my-webapp.local" | sudo tee -a /etc/hosts
```

In EKS, the domain is real. The flow is:
```
Route53 (DNS) → ALB (public IP) → Ingress Controller → Service → Pods
```

Your team would have a real domain (e.g. `myapp.company.com`) pointing to the ALB via Route53.

## Path-based and Host-based Routing

One of the key benefits of Ingress over a plain Service — you can route different traffic
to different Services based on domain or path:

```yaml
# Host-based routing — different domains → different services
rules:
  - host: api.myapp.com
    http:
      paths:
        - backend:
            service:
              name: api-service
  - host: app.myapp.com
    http:
      paths:
        - backend:
            service:
              name: frontend-service
```

```yaml
# Path-based routing — same domain, different paths → different services
rules:
  - host: myapp.com
    http:
      paths:
        - path: /api
          backend:
            service:
              name: api-service
        - path: /
          backend:
            service:
              name: frontend-service
```

## TLS / HTTPS

Ingress also handles TLS termination — HTTPS ends at the Ingress, traffic inside the cluster
is plain HTTP. Certificates are stored as Kubernetes Secrets and referenced in the Ingress spec.
In EKS, teams typically use AWS Certificate Manager (ACM) for certificates.

## Key Takeaway

As tech lead, when someone says "we need to expose a new service" — that means:
1. Create a Service for the new app (if not already there)
2. Add an Ingress rule pointing the right domain/path to that Service
3. In EKS, the AWS Load Balancer Controller automatically provisions the ALB

The Ingress Controller is your team's DevOps concern. The Ingress rules are your app team's concern.
