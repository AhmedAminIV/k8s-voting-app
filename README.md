# k8s-voting-app

# Kubernetes Voting App

This repo contains the **Example Voting App** deployed on Kubernetes, with:
- Frontend apps (`vote`, `result`)
- Backend services (`worker`, `redis`, `postgres`)
- Ingress with TLS
- NetworkPolicies for traffic isolation

---

## 🚀 Features
- `/vote` – voting frontend
- `/result` – results frontend
- **Worker** service processes votes from Redis → Postgres
- **Ingress Controller (NGINX)** with TLS (self-signed certificate)
- **NetworkPolicies** to restrict traffic:
  - Only `vote` can access `redis`
  - Only `worker` can access `postgres`
  - Other cross-namespace traffic blocked

---

## 📂 Repo Structure
```

manifests/
├── namespace.yaml
├── vote-deploy.yaml
├── result-deploy.yaml
├── worker-deploy.yaml
├── redis-deploy.yaml
├── postgres-deploy.yaml
├── services.yaml
├── ingress.yaml
├── tls-secret.yaml
├── networkpolicies.yaml

````

---

## ⚡ Deploy

```bash
kubectl apply -f manifests/
````

Check resources:

```bash
kubectl get pods -n my-app
kubectl get svc -n my-app
kubectl get ingress -n my-app
```

---

## 🌍 Accessing the App

Since Killercoda/Playgrounds do not provide LoadBalancers, use one of:

### Option 1: Port-forward

```bash
kubectl port-forward svc/vote -n my-app 8081:8081
kubectl port-forward svc/result -n my-app 8080:8080
```

Then open in browser:

* [http://localhost:8081/vote](http://localhost:8081/vote)
* [http://localhost:8080/result](http://localhost:8080/result)

### Option 2: Expose Ingress via NodePort

Patch ingress controller service:

```bash
kubectl edit svc ingress-nginx-controller -n ingress-nginx
```

Change type: `LoadBalancer` → `NodePort`.
Use the provided URL (Killercoda shows `*.spch.r.killercoda.com`).

---

## 🔒 TLS

A self-signed certificate is included:

```bash
kubectl create secret tls vote-tls \
  --key tls.key \
  --cert tls.crt \
  -n my-app
```

Ingress routes `/vote` and `/result` through HTTPS.

---

## 🛡️ Network Policies

Example rules included:

* **vote → redis** (allow)
* **worker → postgres** (allow)
* **deny all else**

```bash
kubectl get networkpolicy -n my-app
```

---

## 📸 Screenshots

(Add screenshots from Killercoda browser preview)

```

---
```
