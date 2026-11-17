# ☸️ Kubernetes Lab — Multi-Node Cluster, Deployments & Rollbacks

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.37.0-326CE5?logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Desktop-2496ED?logo=docker&logoColor=white)
![Kind](https://img.shields.io/badge/Kind-Local%20Cluster-326CE5?logo=kubernetes&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Windows%2011-0078D6?logo=windows&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success)

A hands-on Kubernetes laboratory focused on deploying and managing containerized applications in a local multi-node cluster using **Kind (Kubernetes in Docker)**.

This project demonstrates Kubernetes cluster architecture, declarative resource management, application exposure, self-healing, horizontal scaling, rolling updates, and rollback mechanisms.

The lab was performed on **Windows 11 using WSL2 and Docker Desktop**, as part of the *Virtualization and Cloud Computing* module at ENSA Al Hoceima.

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Technologies & Tools](#-technologies--tools)
- [Repository Structure](#-repository-structure)
- [Lab Implementation](#-lab-implementation)
  - [1. Multi-Node Cluster Creation](#1-multi-node-cluster-creation)
  - [2. Pod Management](#2-pod-management)
  - [3. Deployment and Service](#3-deployment-and-service)
  - [4. Self-Healing and Scaling](#4-self-healing-and-scaling)
  - [5. Rolling Update](#5-rolling-update)
  - [6. Failure Simulation and Rollback](#6-failure-simulation-and-rollback)
- [Getting Started](#-getting-started)
- [Verification Commands](#-verification-commands)
- [Key Concepts Learned](#-key-concepts-learned)
- [Author](#-author)

---

## 🎯 Project Overview

The main objective of this lab is to understand how Kubernetes orchestrates containerized applications and ensures their availability.

The practical work covers:

- Creating a local Kubernetes cluster with **3 nodes**:
  - 1 Control Plane
  - 2 Worker Nodes
- Understanding Kubernetes system components and node responsibilities.
- Creating and inspecting standalone Pods.
- Deploying applications declaratively using YAML manifests.
- Managing application replicas through Deployments and ReplicaSets.
- Exposing applications through NodePort Services.
- Testing automatic Pod recovery and horizontal scaling.
- Performing rolling updates without intentionally interrupting application availability.
- Simulating a deployment failure using a non-existent container image.
- Restoring a healthy application version using Kubernetes rollback mechanisms.

The deployed application is **Traefik Whoami**, a lightweight HTTP application that displays information about the Pod handling each request.

---

## 🏗️ Architecture

### Local Environment

The Kubernetes environment is built using the following architecture:

```text
                Windows 11
                    |
          PowerShell / Terminal
                    |
                   WSL2
                    |
              Docker Desktop
                    |
             Kind Kubernetes
                    |
          +---------+---------+
          |                   |
    Control Plane         Worker Nodes
                          /          \
                     Worker 1      Worker 2
```

### Kubernetes Cluster

| Component | Configuration |
|---|---|
| Cluster name | `tp-k8s` |
| Kubernetes distribution | Kind |
| Kubernetes version | v1.37.0 |
| Number of nodes | 3 |
| Control Plane | 1 |
| Worker Nodes | 2 |
| Container runtime | containerd |
| CNI | kindnet |

### Application Architecture

The application follows the Kubernetes resource hierarchy:

```text
             Deployment: whoami
                (6 replicas)
                     |
                     v
                 ReplicaSet
                     |
          +----------+----------+
          |          |          |
         Pod        Pod        Pod
          |          |          |
          +----------+----------+
                     |
                     v
              Service: whoami
                 NodePort
                 Port 30080
                     |
                     v
             http://localhost:30080
```

The Deployment manages the ReplicaSet, which maintains the desired number of Pods. The Service discovers healthy Pods through label selectors and distributes incoming requests between them.

---

## 🛠️ Technologies & Tools

| Technology | Purpose |
|---|---|
| Kubernetes | Container orchestration |
| Kind | Local multi-node Kubernetes cluster |
| Docker Desktop | Container runtime environment |
| Docker | Container management |
| kubectl | Kubernetes cluster administration |
| WSL2 | Linux environment on Windows |
| YAML | Declarative Kubernetes manifests |
| Traefik Whoami | Demo HTTP application |
| PowerShell | Command-line environment |

---

## 📁 Repository Structure

```text
k8s-lab/
│
├── README.md
│
└── lab4/
    ├── kind-config.yaml
    ├── pod.yaml
    ├── whoami.yaml
    ├── rollout-history.txt
    ├── rollout-history-apres-v1.11.txt
    │
    └── captures/
        └── *.txt
```

### File Description

| File | Description |
|---|---|
| `kind-config.yaml` | Defines the 3-node Kind cluster and port mapping. |
| `pod.yaml` | Declarative manifest for the standalone Nginx Pod. |
| `whoami.yaml` | Defines the Whoami Deployment and NodePort Service. |
| `rollout-history.txt` | Final Deployment rollout history after rollback. |
| `rollout-history-apres-v1.11.txt` | Rollout history after the successful v1.11 update. |
| `captures/` | Contains command outputs and verification evidence. |

---

## 🚀 Lab Implementation

### 1. Multi-Node Cluster Creation

Created a local Kubernetes cluster named `tp-k8s` using Kind.

The cluster consists of one Control Plane and two Worker Nodes.

The configuration also maps port `30080` from the host machine to the Control Plane node, allowing access to the application through localhost.

```bash
kind create cluster --config lab4/kind-config.yaml
```

Verify the cluster:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
docker ps
```

**Expected result:** Three Kubernetes nodes in `Ready` state.

### 2. Pod Management

Created a standalone Nginx Pod to explore the basic Kubernetes Pod lifecycle.

```bash
kubectl run web --image=nginx:1.27 --port=80

kubectl get pods -o wide
kubectl describe pod web
kubectl logs web
kubectl exec -it web -- sh
```

Access the application using port forwarding:

```bash
kubectl port-forward pod/web 8080:80
```

Open:

`http://localhost:8080`

The experiment also demonstrates that deleting a standalone Pod does not automatically recreate it because no controller manages its desired state.

The same Pod was subsequently recreated using a declarative YAML manifest:

```bash
kubectl apply -f lab4/pod.yaml
```

### 3. Deployment and Service

Deployed the Traefik Whoami application using a Kubernetes Deployment with three initial replicas.

The `whoami.yaml` manifest contains two resources:

- **Deployment:** Manages the application Pods.
- **Service:** Exposes the application using NodePort.

```bash
kubectl apply -f lab4/whoami.yaml
```

Inspect the deployed resources:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
kubectl get services
```

Access the application:

```bash
curl.exe http://localhost:30080
```

The Service exposes the application on port `30080` and forwards requests to the application Pods on port `80`.

### 4. Self-Healing and Scaling

Tested Kubernetes self-healing by manually deleting one of the Pods managed by the Deployment.

```bash
kubectl get pods
kubectl delete pod <pod-name>
kubectl get pods
```

The ReplicaSet automatically creates a replacement Pod to maintain the desired replica count.

Then scaled the application from three to six replicas:

```bash
kubectl scale deployment whoami --replicas=6
```

Verify:

```bash
kubectl get deployment whoami
kubectl get pods
```

**Expected result:** Six running application Pods.

### 5. Rolling Update

Performed an application update from `traefik/whoami:v1.10` to `traefik/whoami:v1.11`.

```bash
kubectl set image deployment/whoami whoami=traefik/whoami:v1.11
```

Monitor the update:

```bash
kubectl rollout status deployment/whoami
```

Inspect the rollout history:

```bash
kubectl rollout history deployment/whoami
```

Kubernetes progressively replaces old Pods with new ones using the RollingUpdate strategy.

This mechanism helps maintain application availability during updates.

### 6. Failure Simulation and Rollback

Simulated a deployment failure by attempting to deploy a non-existent image version:

```bash
kubectl set image deployment/whoami whoami=traefik/whoami:v9.99
```

Inspect the Pods:

```bash
kubectl get pods
```

The new Pods enter an `ErrImagePull` state because the requested image does not exist.

However, the healthy application replicas continue serving requests.

Restored the previous working version using:

```bash
kubectl rollout undo deployment/whoami
```

Verify the recovery:

```bash
kubectl rollout status deployment/whoami
kubectl get replicasets
kubectl get pods
kubectl rollout history deployment/whoami
```

**Expected result:** The Deployment returns to the healthy `v1.11` version, with six running replicas.

---

## ⚙️ Getting Started

### Prerequisites

Make sure the following tools are installed and configured:

- Windows 11
- WSL2 with Ubuntu
- Docker Desktop
- Kind
- kubectl

### 1. Clone the Repository

```bash
git clone https://github.com/asmaaaa01ab/k8s-lab.git
cd k8s-lab
```

### 2. Create the Kubernetes Cluster

```bash
kind create cluster --config lab4/kind-config.yaml
```

### 3. Deploy the Application

```bash
kubectl apply -f lab4/whoami.yaml
```

### 4. Verify the Deployment

```bash
kubectl get nodes
kubectl get deployments
kubectl get pods -o wide
kubectl get services
```

### 5. Access the Application

Open your browser or execute:

```bash
curl.exe http://localhost:30080
```

### 6. Clean Up

Delete the Kubernetes resources and the local cluster:

```bash
kubectl delete -f lab4/whoami.yaml
kind delete cluster --name tp-k8s
```

---

## 🔍 Verification Commands

Useful commands for inspecting and troubleshooting the cluster:

| Command | Purpose |
|---|---|
| `kubectl get nodes` | List cluster nodes |
| `kubectl get pods -A` | List Pods across namespaces |
| `kubectl describe pod <name>` | Inspect Pod details and events |
| `kubectl logs <pod-name>` | Display container logs |
| `kubectl get deployments` | List Deployments |
| `kubectl get replicasets` | List ReplicaSets |
| `kubectl get services` | List Services |
| `kubectl get endpointslices` | Inspect Service endpoints |
| `kubectl rollout status deployment/whoami` | Monitor deployment updates |
| `kubectl rollout history deployment/whoami` | Display deployment history |
| `kubectl rollout undo deployment/whoami` | Roll back to the previous revision |

---

## 📚 Key Concepts Learned

Through this practical laboratory, I explored and implemented the following Kubernetes concepts:

- **Cluster Architecture:** Control Plane, Worker Nodes, API Server, etcd, Scheduler and Controller Manager.
- **Pod Lifecycle:** Pod creation, inspection, execution and deletion.
- **Declarative Management:** Defining and applying Kubernetes resources using YAML manifests.
- **Deployments and ReplicaSets:** Managing application replicas and maintaining the desired state.
- **Services and Networking:** Exposing applications through NodePort and understanding label selectors and EndpointSlices.
- **Self-Healing:** Automatically replacing deleted Pods.
- **Scaling:** Increasing and decreasing application replicas.
- **Rolling Updates:** Updating application versions progressively.
- **Deployment Strategies:** Understanding `maxSurge` and `maxUnavailable`.
- **Rollback and Recovery:** Recovering from unsuccessful deployments using rollout history and `kubectl rollout undo`.
- **Troubleshooting:** Identifying image-pull failures and inspecting Kubernetes events and resource states.

---

*This repository documents a practical Kubernetes learning experience, from local cluster provisioning to application deployment, scaling, failure handling and recovery.*
