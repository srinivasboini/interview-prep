# Kubernetes and Container Orchestration

## Overview

If Docker revolutionized how we package a single application, Kubernetes (K8s) revolutionized how we run thousands of them. Originally developed by Google (based on their internal Borg system), Kubernetes is now the undisputed operating system of the cloud. It abstracts away the underlying hardware (AWS, Azure, or on-premise bare metal) and provides a unified, declarative API to deploy, scale, and manage containerized applications.

For a Staff/Principal Engineer, especially in banking where hybrid clouds (on-prem + public cloud) are common, Kubernetes is the great equalizer. Interviewers will not ask you how to run a `kubectl` command. They will expect you to architect for Kubernetes: understanding how to manage stateful databases (StatefulSets), how network traffic routes from the public internet to a specific container (Ingress -> Service -> Pod), and how to configure resource limits to prevent a runaway Java application from crashing a shared node.

If you deploy a Spring Boot application to Kubernetes without configuring Liveness/Readiness probes and CPU/Memory limits, you are deploying a ticking time bomb.

## Foundational Concepts

### The Control Plane vs. Compute Nodes (Data Plane)

A Kubernetes cluster is divided into two distinct halves:
1.  **The Control Plane (Master Nodes)**: The brain of the cluster.
    *   **API Server (`kube-apiserver`)**: The only component that interacts directly with the cluster state. All `kubectl` commands and internal communications go through here.
    *   **etcd**: A highly available, consistent Key-Value store holding the entire state of the cluster (what is running, where, and what the desired state is).
    *   **Scheduler (`kube-scheduler`)**: Watches for newly created Pods that have no Node assigned. It selects the optimal Node for them to run on based on resource availability, hardware constraints, and affinity/anti-affinity rules.
    *   **Controller Manager (`kube-controller-manager`)**: Runs control loops that continuously monitor the actual state of the cluster and push it toward the desired state (e.g., if a Deployment asks for 3 replicas and one dies, the ReplicaSet controller notices and requests a new one).

2.  **The Compute Nodes (Worker Nodes)**: The machines (VMs or bare metal) where the actual application workloads run.
    *   **Kubelet**: An agent running on every Node. It listens to the API Server and ensures that the containers described in the Pod specs are actually running and healthy on its machine.
    *   **Kube-proxy**: Manages the network rules on the Node, allowing traffic to route to the correct Pods.
    *   **Container Runtime**: The software that actually runs the containers (e.g., containerd, CRI-O, Docker).

## Technical Deep Dive: Core Kubernetes Objects

Kubernetes operates on declarative YAML manifests. You tell it *what* you want, not *how* to do it.

### 1. Pods
The smallest, most basic unit in Kubernetes. You rarely deploy a container directly; you deploy a Pod. A Pod represents a single instance of a running process in your cluster.
*   *Crucial Detail*: A Pod can contain one or more containers (e.g., the main Java app and a sidecar proxy). All containers in a Pod share the same IP address, localhost network, and storage volumes.
*   *Ephemeral Nature*: Pods are mortal. They are created, they die, and they are never resurrected. If a node fails, the Pods on it are permanently deleted, and the Controller schedules *new* Pods with completely different IP addresses on healthy nodes.

### 2. Workload Controllers (Managing Pod Lifecycle)
You almost *never* create standalone Pods. You use Controllers to manage them.
*   **Deployment**: The most common object for stateless applications (like a REST API). You define a Deployment (saying "I want 5 replicas of the `user-service:v2` image"). The Deployment creates a **ReplicaSet**, which spins up the 5 Pods. If you change the image to `v3`, the Deployment automatically performs a rolling update, gracefully spinning up new Pods and terminating old ones without downtime.
*   **StatefulSet**: Used for stateful applications (like databases or message queues). Unlike a Deployment where Pod IDs are random (e.g., `web-8a4b6c-xh9f`), a StatefulSet gives Pods sticky, predictable network identities (e.g., `mysql-0`, `mysql-1`). If `mysql-1` crashes, it is rescheduled on a new node but retains the exact same name and automatically reattaches to its specific persistent storage volume.
*   **DaemonSet**: Ensures that exactly one copy of a specific Pod runs on *every single node* in the cluster. Used for system-level background tasks: log shippers (Fluentd), monitoring agents (Prometheus Node Exporter), or networking plugins.

### 3. Services (Internal Networking)
Because Pod IP addresses change constantly (due to crashes or scaling), you cannot hardcode them. A **Service** is a stable abstraction (a permanent IP address and DNS name) that groups a set of Pods together using Label Selectors.
*   **ClusterIP (Default)**: Exposes the Service on an internal cluster IP. It is only reachable from within the cluster. Used for microservice-to-microservice communication (e.g., `http://payment-service-clusterip:8080`).
*   **NodePort**: Exposes the Service on the IP of *every* Worker Node at a specific static port (between 30000-32767). An external client hitting `NodeIP:30000` is routed into the cluster. Rarely used directly in production due to security and scaling limitations.
*   **LoadBalancer**: Cloud-provider specific. Automatically provisions an external AWS ELB/Azure Load Balancer that routes external traffic down into the internal ClusterIP.

### 4. Ingress (External Routing)
A LoadBalancer Service provides a 1:1 mapping (1 Public IP -> 1 Kubernetes Service). This gets expensive. An **Ingress** is an API object that manages external access to the services in a cluster, typically HTTP. It acts as a single point of entry (a Layer 7 router), evaluating URL paths or hostnames and routing traffic to different internal Services (e.g., `bank.com/api/users` routes to `user-svc`, while `bank.com/api/payments` routes to `payment-svc`).

## Visual Representations

### The Kubernetes Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#E3F2FD', 'edgeLabelBackground':'#FFF'}}}%%
flowchart TD
    classDef control fill:#E1BEE7,stroke:#8E24AA,stroke-width:2px;
    classDef worker fill:#C8E6C9,stroke:#388E3C,stroke-width:2px;
    classDef network fill:#FFF9C4,stroke:#FBC02D,stroke-width:2px;
    classDef storage fill:#FFCCBC,stroke:#E64A19,stroke-width:2px;

    Client((External \n User))

    subgraph Control Plane (Master Nodes)
        API[kube-apiserver]:::control
        ETCD[(etcd State Store)]:::control
        Sched[kube-scheduler]:::control
        CM[kube-controller-manager]:::control
        
        API <--> ETCD
        Sched -->|Assigns Pods| API
        CM -->|Reconciles State| API
    end

    Client -->|HTTPS / API| LB[Cloud Load Balancer]:::network
    LB --> INGRESS[Ingress Controller \n NGINX/Envoy]:::network

    subgraph Worker Node 1
        K1[Kubelet]:::worker
        KP1[Kube-proxy]:::worker
        
        subgraph Pod: User Service
            C1[Java Container]:::worker
        end
    end

    subgraph Worker Node 2
        K2[Kubelet]:::worker
        KP2[Kube-proxy]:::worker
        
        subgraph Pod: DB Node 0
            C2[Postgres Container]:::worker
        end
        PV[(Persistent Volume)]:::storage
    end

    API -.->|Instructions| K1
    API -.->|Instructions| K2
    
    INGRESS -->|Routes Traffic| KP1
    KP1 --> C1
    
    C1 -->|Internal ClusterIP| KP2
    KP2 --> C2
    C2 ----> PV
```

## Code/Configuration Examples

### Enterprise-Grade Deployment YAML (Java/Spring Boot)

This manifest demonstrates how a Staff Engineer configures a production Deployment. 
*Notice the strict resource limits, health probes, and anti-affinity rules.*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    app: payment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  # Ensure zero-downtime rolling updates
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # Spin up max 1 new pod before killing an old one
      maxUnavailable: 0 # Never dip below 3 healthy pods during rollout
  template:
    metadata:
      labels:
        app: payment
    spec:
      # Pod Anti-Affinity: Mathematically prevent the scheduler from putting 
      # multiple payment-service pods on the exact same physical Worker Node.
      # If an AWS EC2 instance physically burns, we only lose 1 pod, not 3.
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - payment
            topologyKey: "kubernetes.io/hostname"
            
      containers:
      - name: payment-app
        image: custom-registry.bank.internal/payment-service:v2.1.0
        ports:
        - containerPort: 8080
        
        # Resource Requests & Limits: CRITICAL for Java
        # Requests: What the scheduler guarantees (Used for node placement).
        # Limits: Hard ceiling. If Java exceeds this memory, Linux OOMKills the container instantly.
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m" # Half a core
          limits:
            memory: "1024Mi"
            cpu: "1000m" # 1 full core
            
        # Startup Probe: Java/Spring Boot takes a long time to start. 
        # Don't kill it prematurely while it's booting up.
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          failureThreshold: 30
          periodSeconds: 10 # Gives Spring Boot up to 5 minutes (30 * 10s) to boot
          
        # Liveness Probe: Detects if the app is deadlocked. 
        # If this fails, Kubelet restarts the container immediately.
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          
        # Readiness Probe: Detects if the app is ready to take network traffic.
        # If this fails (e.g., DB connection dropped), traffic stops routing to this pod, 
        # but the pod is NOT restarted.
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

## Interview Questions & Model Answers

**Q1: We deployed a Java Spring Boot application to Kubernetes. Sometimes, nodes randomly crash with OutOfMemory (OOM) errors, taking down unrelated pods running on that same node. How do we fix this?**
*Answer*: This is the classic "Noisy Neighbor" problem caused by deploying pods without defining resource `limits`. By default, a container in Kubernetes can consume unlimited CPU and Memory from the underlying host Node. A JVM with unbounded memory will continuously expand its heap until the underlying Linux kernel runs out of RAM and aggressively kills processes to survive.
To fix this, we must configure strict `requests` and `limits` in the Deployment manifest.
*   **Requests** guarantee that the `kube-scheduler` will only place the pod on a Node that actually has that much free memory.
*   **Limits** act as a hard ceiling. If the JVM tries to allocate more memory than its limit (e.g., 1024Mi), the container runtime (`containerd`) will OOMKill the *container*, instantly restarting it. This isolates the failure to the specific pod preventing it from choking the entire Node and crashing other microservices. Additionally, we must configure the JVM flags (e.g., `-XX:MaxRAMPercentage=75.0`) to make Java aware of the container's limits.

**Q2: Contrast Liveness and Readiness Probes. What happens if you use a Liveness probe to check the database connection?**
*Answer*: 
*   A **Liveness Probe** determines if the container's main process is deadlocked or permanently crashed. If it fails, Kubernetes will immediately `kill -9` the container and restart it.
*   A **Readiness Probe** determines if the application is currently capable of handling HTTP network traffic. If it fails, Kubernetes simply removes the Pod from the Service load balancer so it stops receiving traffic; it does *not* restart the container.
If you use a Liveness probe to check a database connection (e.g., `SELECT 1`), and the database goes down for 30 seconds for routine maintenance, every single microservice pod connected to that database will fail its Liveness probe. Kubernetes will slaughter hundreds of healthy application pods and repeatedly try to restart them in a massive cluster-wide restart storm. Only use Readiness probes for checking external downstream dependencies. Liveness probes should only check internal application health (e.g., "Is my HTTP server thread pool responding?").

**Q3: We need to run a high-throughput Apache Kafka cluster inside Kubernetes. Should we use a Deployment or a StatefulSet? Provide architectural reasoning.**
*Answer*: You must use a **StatefulSet**. A `Deployment` assumes all pods are completely stateless, identical, and interchangeable. If a deployment pod crashes, the replacement pod spins up on a new node, gets a new random hash ID, and loses connection to the previous pod's disk.
Kafka brokers are not identical. Broker 1 holds Partition A, while Broker 2 holds Partition B.
A StatefulSet provides three critical guarantees for databases/message queues:
1.  **Sticky Identity**: Pods get predictable, ordered names (`kafka-0`, `kafka-1`, `kafka-2`). If `kafka-1` crashes, the replacement pod is explicitly named `kafka-1` and retains its network DNS identity.
2.  **Stable Storage**: When `kafka-1` is created, a PersistentVolumeClaim (PVC) is attached to it. If the pod dies and moves to a different node, Kubernetes natively detaches the massive disk from the old node and reattaches it to the new node, ensuring data is not lost.
3.  **Ordered Deployment**: Pods spin up sequentially (`0`, then `1`, then `2`). This is required for clustered leader election protocols (like ZooKeeper/Kraft) to form a quorum safely.

**Q4: Explain the role of an Ingress Controller in a microservices architecture. Why wouldn't we just give every microservice a LoadBalancer Service?**
*Answer*: If you have 50 microservices and give each a Service type of `LoadBalancer`, Kubernetes will automatically provision 50 separate Public IP load balancers in AWS/Azure. This costs a fortune and creates a fragmented, impossible-to-manage security perimeter.
An **Ingress Controller** (like NGINX or Traefik) is a Layer 7 (HTTP/HTTPS) reverse proxy running *inside* the cluster, usually backed by just a single external LoadBalancer. You write declarative Ingress rules that instruct the controller on how to route external traffic internally. For example, `bank.com/api/users` routes to the internal `user-svc` ClusterIP, and `bank.com/api/payments` routes to `payment-svc`. Furthermore, the Ingress Controller acts as the single point for SSL Termination, meaning all internal microservice-to-microservice traffic can optionally run over unencrypted HTTP (improving CPU efficiency) while external traffic remains heavily secured with TLS certificates managed in one central location.

## Real-World Enterprise Scenarios

**Scenario: Blue/Green Deployment with Kubernetes Services**
*   **Context**: Deploying a massive rewrite of the `LedgerService` without bringing the bank down.
*   **Architecture**:
    1.  The production traffic points to a Service named `ledger-live`. This Service uses a label selector `app: ledger, version: v1`. It routes traffic to the 10 pods running V1.
    2.  We deploy a brand new Deployment specifying the V2 image, with labels `app: ledger, version: v2`. We also deploy a temporary Service `ledger-test` pointing only to V2.
    3.  Internal QA teams hit the `ledger-test` service to verify everything works against the live database.
    4.  **The Cutover**: To release it to customers, we edit the `ledger-live` Service YAML, changing the selector from `version: v1` to `version: v2` and applying it.
    5.  Kubernetes updates the `iptables` routing rules inside the cluster instantly. 100% of traffic immediately shifts to the V2 pods. Zero downtime. If it errors, we change the label back to `v1` instantly.

## Common Pitfalls & Best Practices

**Pitfalls:**
*   **Treating Pods like VMs**: SSHing into a running Pod to change a configuration file or restart a service. The moment the Pod scales down or crashes, all those manual changes are obliterated forever. Configure via ConfigMaps, track in Git.
*   **Using `latest` image tags**: A Deployment defining `image: my-app:latest` forces Kubelet to constantly pull the image to see if it changed. Worse, if an incident occurs and 50 pods restart, they will pull whatever developer pushed exactly at that minute, meaning you now have 50 pods running completely untested, unapproved code in production. Always mandate strict, immutable Semantic Versioning (`v1.4.2`).
*   **Missing PodDisruptionBudgets (PDB)**: If a cluster administrator drains a node for an OS upgrade, Kubernetes will evict all pods on that node. If your application has 3 replicas, and the admin upgrades 3 nodes simultaneously, your application suffers a total outage. A PDB (`minAvailable: 2`) instructs Kubernetes it is mathematically forbidden from draining nodes if it drops the healthy pod count for that app below 2, forcing the upgrade to happen safely and sequentially.

**Best Practices:**
*   **Horizontal Pod Autoscaling (HPA)**: Do not scale pods manually. Configure an HPA object targeting 70% CPU usage. If traffic spikes and CPU hits 80%, the HPA alerts the Controller Manager to increase the Replica count of the Deployment from 3 to 10. When traffic subsides, it scales back down to save infrastructure costs.
*   **Graceful Shutdown**: When Kubernetes terminates a pod (e.g., during scale-down), it sends a `SIGTERM` signal. The Java application has 30 seconds (by default) to finish processing active HTTP requests and gracefully close database connection pools before Kubernetes brutally executes a `SIGKILL` (`kill -9`). Ensure Spring Boot graceful shutdown is enabled (`server.shutdown=graceful`).
*   **Externalize Configuration via Secrets/ConfigMaps**: Never embed database passwords or API keys inside the Docker Image or the Deployment YAML. Store them as Kubernetes `Secret` objects. Pass them securely into the Pods as Environment Variables or mounted volumes at runtime.

## Comparison Tables

| Kubernetes Object | Use Case | State Persistence | Predictable Network Identity |
| :--- | :--- | :--- | :--- |
| **Deployment / ReplicaSet**| REST APIs, Stateless workers, Web UIs | Ephemeral (Lost on restart) | No (Random hashes appended) |
| **StatefulSet** | Databases, Kafka, Elasticsearch, Redis | Persistent (PVC survives restarts)| Yes (e.g., `mysql-0`) |
| **DaemonSet** | Log shippers (Fluentd), Monitoring agents| Per-Node persistence | Bound to specific physical nodes |
| **Job / CronJob** | Nightly batch processing, DB migrations | Results stored externally | Ephemeral execution |

| Service Type | Accessibility | Use Case | Cost Profile |
| :--- | :--- | :--- | :--- |
| **ClusterIP** | Internal to Cluster Only | Microservice -> Microservice comms | Free |
| **NodePort** | External (over specific high port) | Testing, simple legacy integrations | Free, but complex firewall management |
| **LoadBalancer** | External (Standard Ports 80/443) | Monolithic apps, simple endpoints | High (Provisions Cloud Hardware Load Balancer) |
| **Ingress (L7)** | External (Path/Host routing) | Managing 50+ microservice endpoints | Low (1 Load Balancer serving many paths) |

## Key Takeaways

*   **Kubernetes is Declarative**: You write the YAML manifest outlining the "Desired State". The Master Control Plane loops infinitely, adjusting the infrastructure until "Actual State == Desired State".
*   **Pods are ephemeral cattle**: Assume any Pod can be destroyed instantly. Build stateless HTTP APIs attached to Deployment objects. Manage heavy lifting state in external databases, or carefully configure StatefulSets.
*   **Probes dictate Survival**: If you misconfigure a Liveness Probe to check a slow downstream database, Kubernetes will endlessly massacre healthy containers.
*   **Strict Resource Boundaries**: Limits and Requests prevent runaway JVMs from crashing shared enterprise infrastructure. Do not deploy to prod without them.

## Further Reading
*   [Kubernetes Components (Official Documentation)](https://kubernetes.io/docs/concepts/overview/components/)
*   [Kubernetes Best Practices: Resource Requests and Limits](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-resource-requests-and-limits)
*   [Managing State with StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
*   [Liveness and Readiness Probes (Spring Boot Integration)](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot)
