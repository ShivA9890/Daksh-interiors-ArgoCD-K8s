# Kubernetes Deployment Guide

> Complete guide for deploying the Daksh application on a local Kubernetes cluster using **Kind**, **NGINX Ingress**, **Horizontal Pod Autoscaler (HPA)**, **Prometheus**, and **Grafana**.

---

# Table of Contents

* Project Architecture
* Prerequisites
* Install Kind
* Install kubectl
* Create Kubernetes Cluster
* Create Namespace
* Deploy Applications
* Create Services
* Install NGINX Ingress Controller
* Configure Ingress
* Access the Application
* Configure Horizontal Pod Autoscaler (HPA)
* Install Metrics Server
* Configure Metrics Server
* Deploy HPA
* Install Prometheus & Grafana
* Useful kubectl Commands
* Local Ports
* Troubleshooting
* Complete Deployment Flow

---

# Project Architecture

```text
                    +----------------------+
                    |      Browser         |
                    +----------+-----------+
                               |
                      http://localhost:8081
                               |
                    NGINX Ingress Controller
                               |
                +--------------+--------------+
                |                             |
          Client Service                 Server Service
                |                             |
        Client Deployment             Server Deployment
                |                             |
             Client Pods                  Server Pods
                \_______________________________/
                                |
                        Kubernetes Cluster
                                |
                               Kind
                                |
                         Docker Desktop
```

---

# Prerequisites

Before starting, install the following software:

| Software       | Purpose                                    |
| -------------- | ------------------------------------------ |
| Docker Desktop | Runs Kubernetes nodes as Docker containers |
| Kind           | Creates local Kubernetes cluster           |
| kubectl        | Kubernetes CLI                             |
| Helm           | Kubernetes Package Manager                 |

---

# 1. Install Kind

```bash
brew install kind
```

Verify installation

```bash
kind version
```

## Why do we use Kind?

Kind (Kubernetes IN Docker) creates a complete Kubernetes cluster using Docker containers.

Advantages:

* Lightweight
* Fast startup
* Perfect for local development
* Great for CI/CD testing
* No Virtual Machine required

---

# 2. Install kubectl

Verify installation

```bash
kubectl version --client
```

## Why?

kubectl is the command-line interface used to communicate with Kubernetes.

Using kubectl you can:

* Deploy applications
* Scale deployments
* View logs
* Debug pods
* Create services
* Monitor cluster resources

Without kubectl you cannot manage Kubernetes resources.

---

# 3. Create Kubernetes Cluster

```bash
kind create cluster --config kind-config.yaml --name daksh-int
```

## What happens?

This creates a Kubernetes cluster using the configuration defined inside:

```
kind-config.yaml
```

The configuration generally defines

* Cluster name
* Node configuration
* Port mapping
* Networking
* Control Plane settings

Verify

```bash
kubectl cluster-info
```

---

# 4. Create Namespace

```bash
kubectl apply -f namespace.yaml
```

## Why?

Namespaces logically separate Kubernetes resources.

Instead of keeping everything inside the **default** namespace, we create

```
daksh-in
```

Benefits

* Better organization
* Easier debugging
* Environment isolation
* Resource separation

Verify

```bash
kubectl get namespaces
```

---

# 5. Deploy Applications

```bash
kubectl apply -f deployments-client.yaml -f deployments-server.yaml -n daksh-in
```

## What is a Deployment?

A Deployment manages Pods.

Instead of creating Pods manually, Deployments provide

* Self healing
* Rolling updates
* Replica management
* Automatic restart
* Scaling

Your project contains

* Client Deployment
* Server Deployment

Verify

```bash
kubectl get deployments -n daksh-in
```

Check Pods

```bash
kubectl get pods -n daksh-in
```

---

# 6. Create Services

```bash
kubectl apply -f service-internal.yaml -f service-external.yaml -n daksh-in
```

## Why are Services required?

Pods receive dynamic IP addresses.

Whenever a Pod restarts, its IP changes.

Services provide

* Stable IP
* Stable DNS
* Load balancing
* Internal communication

Architecture

```
Browser
      ↓
Ingress
      ↓
Service
      ↓
Pods
```

Verify

```bash
kubectl get svc -n daksh-in
```

---

# 7. Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait until all pods are Running.

```bash
kubectl get pods -n ingress-nginx -w
```

## Why do we need an Ingress Controller?

Without Ingress

```
Client → NodePort
Server → NodePort
API → NodePort
```

Multiple ports must be exposed.

With Ingress

```
localhost

↓

Ingress

↓

Routes traffic automatically
```

Benefits

* Single Entry Point
* Reverse Proxy
* Load Balancing
* SSL Support
* URL Routing

---

# 8. Create Ingress

```bash
kubectl apply -f ingress-service.yaml -n daksh-in
```

## What does Ingress do?

Ingress defines routing rules.

Example

```
localhost

↓

Ingress Controller

↓

Client Service

↓

Client Pods
```

Verify

```bash
kubectl get ingress -n daksh-in
```

---

# 9. Restart Ingress Controller

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

## Why?

Reloads the Ingress Controller so new routing rules are applied.

---

# 10. Run Application

```bash
kubectl port-forward \
-n ingress-nginx \
service/ingress-nginx-controller \
8081:80 \
--address 0.0.0.0
```

Open

```
http://localhost:8081
```

## Why Port Forward?

The Ingress Controller is running inside Kubernetes.

Port forwarding exposes it to your local machine.

---

# Horizontal Pod Autoscaler (HPA)

## Why HPA?

Suppose you have

```
1 Pod
```

Suddenly

```
500 Users
```

One Pod becomes overloaded.

HPA automatically increases Pods.

```
1 Pod

↓

CPU 80%

↓

3 Pods

↓

Traffic Shared
```

Advantages

* Automatic Scaling
* Better Performance
* High Availability
* Cost Optimization

---

# 11. Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Why?

HPA needs CPU and Memory metrics.

Metrics Server collects

* CPU Usage
* Memory Usage

Without Metrics Server

```
kubectl top pods
```

will fail.

---

# 12. Configure Metrics Server

Either edit deployment

```bash
kubectl -n kube-system edit deployment metrics-server
```

or patch directly

```bash
kubectl patch -n kube-system deployment metrics-server --type=json -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname"}]'
```

## Why?

Kind uses self-signed certificates.

These arguments allow Metrics Server to communicate with Kubernetes nodes.

Restart

```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

Verify

```bash
kubectl top nodes
```

---

# 13. Deploy HPA

```bash
kubectl apply -f hpa-client.yaml -f hpa-server.yaml -n daksh-in
```

Verify

```bash
kubectl get hpa -n daksh-in
```

Congratulations!

Your application now scales automatically.

---

# Monitoring using Prometheus & Grafana

Monitoring helps answer questions like

* Which Pod is consuming CPU?
* Which Deployment restarted?
* Memory usage?
* Node health?
* Request rate?
* Application availability?

---

# 14. Install Helm

```bash
brew install helm
```

Verify

```bash
helm version
```

## Why Helm?

Helm is the package manager for Kubernetes.

Instead of manually creating hundreds of YAML files, Helm installs applications with a single command.

---

# 15. Add Prometheus Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update
```

---

# 16. Install Prometheus & Grafana

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

Wait

```bash
kubectl get pods -n monitoring -w
```

This installs

* Prometheus
* Grafana
* AlertManager
* kube-state-metrics
* Node Exporter

---

# Useful kubectl Commands

## Get all Pods

```bash
kubectl get pods -A
```

## Get Pods in Namespace

```bash
kubectl get pods -n daksh-in
```

## Get Services

```bash
kubectl get svc -A
```

## Get Deployments

```bash
kubectl get deployments -A
```

## Get HPA

```bash
kubectl get hpa -A
```

## Describe Pod

```bash
kubectl describe pod <pod-name>
```

## View Logs

```bash
kubectl logs <pod-name>
```

---

# Local Ports

| Port | Service     |
| ---- | ----------- |
| 8080 | Jenkins     |
| 9000 | SonarQube   |
| 8081 | Application |

---

# Troubleshooting

## Application not opening

Check

```bash
kubectl get pods -A
```

Check Ingress

```bash
kubectl get ingress -A
```

Restart Ingress

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

---

## HPA not working

Check Metrics Server

```bash
kubectl get pods -n kube-system
```

Verify

```bash
kubectl top pods
```

---

## Monitoring Pods Pending

Check available memory

```bash
kubectl get pods -n monitoring
```

Prometheus and Grafana consume significant RAM and CPU.

If Docker Desktop has insufficient resources, Pods may continuously restart.

Increase Docker Desktop resources if necessary.

---

# Important Note (Local Development)

Running Prometheus and Grafana locally can consume a large amount of CPU and RAM.

When Docker Desktop runs out of memory, Kubernetes may start evicting Pods. During this process, the NGINX Ingress Controller can restart, causing the localhost port mapping to become inconsistent.

To restore the correct mapping, apply the Ingress patch:

```bash
kubectl apply -f ingress-patch.yaml
```

This ensures that localhost:8081 remains mapped correctly to the Ingress Controller even after restarts.

---

# Complete Deployment Flow

```
Install Docker Desktop
        │
        ▼
Install Kind
        │
        ▼
Install kubectl
        │
        ▼
Create Cluster
        │
        ▼
Create Namespace
        │
        ▼
Deploy Client & Server
        │
        ▼
Create Services
        │
        ▼
Install NGINX Ingress
        │
        ▼
Create Ingress
        │
        ▼
Port Forward
        │
        ▼
Application Running
        │
        ▼
Install Metrics Server
        │
        ▼
Configure Metrics Server
        │
        ▼
Deploy HPA
        │
        ▼
Install Helm
        │
        ▼
Install Prometheus
        │
        ▼
Install Grafana
        │
        ▼
Monitor Cluster
```

---

# Author

**Shiv M**

This project demonstrates deploying a production-like Kubernetes environment locally using **Kind**, **NGINX Ingress**, **Horizontal Pod Autoscaler**, **Prometheus**, and **Grafana**. It is intended for learning Kubernetes concepts, practicing deployments, and building a complete local DevOps workflow.
