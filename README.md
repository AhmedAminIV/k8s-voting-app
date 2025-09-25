# Kubernetes Voting App

This repository provides a complete deployment of the classic **Example Voting App** on a local Kubernetes cluster, managed via **kind** (Kubernetes in Docker). It includes a full microservice architecture, Ingress with TLS termination, and a robust security posture using Network Policies.

---

#### 🗺️ Architecture Diagram
A visual representation of the microservices and their connections within Kubernetes.

![App architecture](/images%20&%20screenshots/architecture.excalidraw.png)

---

## 🚀 Features

The application is deployed to the `my-app` namespace and consists of the following components:

* **`vote` (Frontend)**: The voting web interface.
* **`result` (Frontend)**: The web interface to view current results.
* **`worker` (Backend)**: A process that consumes votes from Redis and persists them into PostgreSQL.
* **`redis` (Backend)**: An in-memory cache for incoming votes.
* **`postgres` (Backend)**: The persistent database for final vote tallies.
* **Ingress Controller (NGINX)**: Provides external access and handles TLS termination with a self-signed certificate.
* **Network Policies**: Configured to restrict traffic for strong service isolation (see **Network Policies** section for details).

---

## 🛠️ Prerequisites

To run this application locally, you must have the following tools installed:

1.  **Docker**: Required to run **kind**.
2.  **kind (Kubernetes in Docker)**: For creating a local Kubernetes cluster.
3.  **kubectl**: The Kubernetes command-line tool.
4.  **openssl**: To generate the self-signed TLS certificate.

---

## ⚙️ Local Setup Instructions

### 1. Create a kind Cluster

Start your local cluster. We'll name it `voting-cluster`.

```bash
kind create cluster --name voting-cluster
````

### 2\. Install the NGINX Ingress Controller

The NGINX Ingress Controller is required to route external traffic to the `vote` and `result` frontends.

**A. Deploy the Controller:**

```bash
# This command applies the standard kind-compatible deployment for the ingress-nginx controller.
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

**B. Expose the Ingress Service:**

By default, the NGINX Ingress controller service is of type `LoadBalancer` which may not be fully supported in a local `kind` cluster. We patch it to use `NodePort` so it's accessible via the host machine's ports.

```bash
# Patch the service to use NodePort for easier access from the host.
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "NodePort"}}'
```

### 3\. Generate TLS Secret

The Ingress resources for `vote.local` and `result.local` require a TLS certificate. We'll generate a self-signed certificate and key.

**A. Generate Certificate and Key:**

```bash
# Generates a 365-day self-signed certificate and key for the domain 'vote.local'.
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout vote.local.key -out vote.local.crt -subj "/CN=vote.local/O=Local Voting App"
```

**B. Create Kubernetes Secret:**

The manifest file `vote-app.yaml` (which should be broken out into smaller files, as suggested in the **Repo Structure** section below) expects the secret to be named **`vote-local-tls`** in the **`my-app`** namespace.

```bash
# The 'my-app' namespace will be created by the main deployment manifest. You may want to run this part again after the 'my-app' namespace is created.
kubectl create secret tls vote-local-tls \
  --cert=vote.local.crt \
  --key=vote.local.key \
  -n my-app --dry-run=client -o yaml | kubectl apply -f -
```

> **Note**: This uses a dry-run/apply method to ensure the command doesn't fail if the namespace doesn't exist yet, as the namespace is part of the deployment file.

-----

## ⚡ Deployment

The application configuration is managed by individual YAML files, ordered using numerical prefixes inside the `manifests/` directory to ensure resources are created in the correct sequence (e.g., Namespace before Deployments, etc.).

1.  **Deploy the Application**:

    Apply all the manifests in the directory with a single command:

    ```bash
    kubectl apply -f manifests/
    ```

2.  **Verify Deployment**:

    Check that all pods, services, and other resources have been created successfully in the `my-app` namespace:

    ```bash
    kubectl get all -n my-app
    kubectl get ingress -n my-app
    kubectl get networkpolicy -n my-app
    ```

    Wait for all Pods to show a `Running` status before proceeding to access the application.
-----

## 🌍 Accessing the Application

Since we are using custom hostnames (`vote.local` and `result.local`), you need to resolve these hostnames to your local machine's IP address.

1.  **Get the NodePort IP and Port:**

    Find the external access port for the NGINX Controller:

    ```bash
    # Get the InternalIP of the Kind control plane node
NODE_IP=$(kubectl get nodes voting-cluster-control-plane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')

# Display the result to verify
echo "Extracted Node IP: $NODE_IP"

    # For kind, the cluster is accessed via the Docker container's IP, but usually, 'localhost' works
    ```

2.  **Update your Hosts File:**

    You must modify your local machine's `/etc/hosts` file (or `C:\Windows\System32\drivers\etc\hosts` on Windows) to map the custom domains to `127.0.0.1` or to the container's IP Address.

    Add the following lines:

    ```bash
    # This command appends the new entries to /etc/hosts, 
    # using the IP retrieved from kubectl.

      echo "$NODE_IP vote.local" | sudo tee -a /etc/hosts
      echo "$NODE_IP result.local" | sudo tee -a /etc/hosts
    ```

3.  **Access the Apps:**

      * **Voting Frontend**: Access via `http://vote.local:$NODE_PORT/`
      * **Results Frontend**: Access via `http://result.local:$NODE_PORT/`

    > **Note**: Because we used a self-signed certificate, your browser will show a security warning. You can safely proceed past the warning to view the site. We use `http` here because the ingress is configured with `nginx.ingress.kubernetes.io/ssl-redirect: "false"`.

-----

## 🛡️ Network Policies Overview

The application is secured with a default-deny policy, and explicit rules are created to allow only necessary traffic, ensuring strong isolation between microservices.

| Source Pod | Destination Pod | Protocol / Port | Purpose | NetworkPolicy Name |
| :--- | :--- | :--- | :--- | :--- |
| `vote` (frontend) | `redis` | TCP / 6379 | Store votes | `allow-vote-to-redis` |
| `result` (frontend) | `postgres` (db) | TCP / 5432 | Read results | `allow-result-to-postgres` |
| `worker` (backend) | `redis` and `postgres` (db) | TCP / 6379 & 5432 | Process and persist votes | `allow-worker-to-db-and-redis` |
| **Ingress-Nginx** | `vote` and `result` | TCP / 80 | External access | `allow-ingress-to-frontends` |

### **To inspect the policies:**

```bash
kubectl get networkpolicy -n my-app
kubectl describe networkpolicy <policy-name> -n my-app
```

-----

## 📂 Repo Structure

This structure places all Kubernetes YAML files into a single `manifests/` directory.

```
kubernetes-voting-app/
├── **README.md** <--
├── **vote.local.key** <-- (The generated self-signed TLS key)
├── **vote.local.crt** <-- (The generated self-signed TLS certificate)
│
├── **images-&-screenshots/**
│
└── **manifests/**
    ├── 00-my-app-namespace.yaml
    ├── 10-vote-deployment.yaml
    ├── 11-vote-svc.yaml
    ├── 12-vote-ingress.yaml
    ├── 20-result-deployment.yaml
    ├── 21-result-svc.yaml
    ├── 22-result-ingress.yaml
    ├── 30-worker-deployment.yaml
    ├── 40-redis-deployment.yaml
    ├── 41-redis-svc.yaml
    ├── 50-postgres-db-deploy.yaml
    ├── 51-postgres-db-svc.yaml
    ├── 60-default-deny-NP.yaml
    ├── 61-frontends-from-nginx-ingress-NP.yaml
    ├── 62-redis-vote-NP.yaml
    ├── 63-postgres-result-NP.yaml
    └── 64-backends-worker-NP.yaml
```

### Deployment Command

The deployment command remains simple and clear:

```bash
kubectl apply -f manifests/
```
-----

## 🧹 Cleanup

When you're finished, you can delete the `kind` cluster:

```bash
kind delete cluster --name voting-cluster
```

```
```



## 📸 Screenshots

![Results Frontend 1](images%20%26%20screenshots/result.jpeg)
![Results Frontend 2](images%20%26%20screenshots/result2)
![Voting Frontend 1](images%20%26%20screenshots/vote.jpeg)
![Voting Frontend 2](images%20%26%20screenshots/vote2)
```

---
```
