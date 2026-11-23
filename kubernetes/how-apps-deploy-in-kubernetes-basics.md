# How Applications Deploy in Kubernetes — Basics

End-to-end flow of an application, from the developer's code to a running container that users can reach.

> Next: [Advanced notes](./how-apps-deploy-in-kubernetes-advanced.md)

The flow has two halves:

- **CI (Continuous Integration):** code → build (compile) → test → publish (artifact) → container image → registry
- **CD / Kubernetes:** manifest → control plane → node → running pod → users

---

## Flow diagram

```mermaid
flowchart TD
    Dev([👨‍💻 Developer]) -->|1. writes & pushes| Code[📄 Code in Git]

    subgraph CI["🔧 CI pipeline (e.g. GitLab CI)"]
        Code -->|2| Build[Restore, compile, unit test]
        Build -->|3| Artifact[📦 Artifact]
        Artifact -->|4| Dockerfile[🐳 Dockerfile]
        Dockerfile -->|5| DBuild[docker build]
        DBuild -->|6| Push[docker push]
    end

    Push -->|7| Registry[(🗄️ Image registry<br/>Docker Hub / Artifact Registry)]

    Dev -->|8. writes| Manifest[📝 K8s Deployment manifest - YAML]
    Manifest -->|9. kubectl apply| API

    subgraph CP["🧠 Control plane"]
        API[kube-apiserver]
        API <-->|10| ETCD[(etcd)]
        CM[controller-manager] <-->|10| API
        SCHED[scheduler] <-->|10| API
        CCM[cloud-controller-manager] <--> API
    end

    subgraph WN["🖥️ Worker node"]
        Kubelet[kubelet]
        Runtime[Container runtime]
        KProxy[kube-proxy]
        Pod[🚀 Pod / container]
    end

    Kubelet -->|11. watches for pods| API
    Kubelet -->|12| Runtime
    Runtime -->|13. pulls image| Registry
    Runtime -->|14. starts| Pod

    Users([👥 Users]) -->|15| Service[🌐 Service]
    Service --> Pod
    KProxy -. programs routing rules .-> Service

    %% ---------- Colours ----------
    classDef dev      fill:#7c3aed,stroke:#4c1d95,color:#ffffff,stroke-width:2px
    classDef ci       fill:#2563eb,stroke:#1e3a8a,color:#ffffff,stroke-width:2px
    classDef registry fill:#0d9488,stroke:#134e4a,color:#ffffff,stroke-width:2px
    classDef cp       fill:#ea580c,stroke:#7c2d12,color:#ffffff,stroke-width:2px
    classDef worker   fill:#16a34a,stroke:#14532d,color:#ffffff,stroke-width:2px
    classDef pod      fill:#facc15,stroke:#a16207,color:#000000,stroke-width:2px
    classDef users    fill:#db2777,stroke:#831843,color:#ffffff,stroke-width:2px

    class Dev,Code,Manifest dev
    class Build,Artifact,Dockerfile,DBuild,Push ci
    class Registry registry
    class API,ETCD,CM,SCHED,CCM cp
    class Kubelet,Runtime,KProxy worker
    class Pod pod
    class Users,Service users

    style CI   fill:#dbeafe,stroke:#2563eb,stroke-width:3px,color:#1e3a8a
    style CP   fill:#ffedd5,stroke:#ea580c,stroke-width:3px,color:#7c2d12
    style WN   fill:#dcfce7,stroke:#16a34a,stroke-width:3px,color:#14532d
```

**Colour legend:** Purple = developer & files · Blue = CI pipeline · Teal = image registry · Orange = control plane · Green = worker node · Yellow = running pod · Pink = users & Service

---

## Step by step

### Part 1: CI (build the image)

| Step | What happens |
|------|--------------|
| 1 | Developer writes code and pushes it to Git. The push triggers the pipeline. |
| 2 | Pipeline **compiles** the code first (`dotnet restore` → `dotnet build`), then runs **unit tests** on the compiled code (`dotnet test`). If a test fails, the pipeline stops here. |
| 3 | After tests pass, `dotnet publish` produces the deployable **artifact**. |
| 4 | The **Dockerfile** (committed in the repo alongside the code) describes how to package the artifact. |
| 5 | `docker build` creates a **container image**. |
| 6–7 | `docker push` uploads the image to a **registry** (Docker Hub, Google Artifact Registry, GitLab Container Registry, etc.). |

> **Order matters:** restore → build (compile) → test → publish → docker build → push.
> Tests need compiled code, so compile comes first. But the **Docker image** is built only *after* tests pass, so a broken build never reaches the registry.

### Part 2: Kubernetes (run the image)

| Step | What happens |
|------|--------------|
| 8 | Developer writes a **Deployment manifest** (YAML): image name, replicas, ports, resources. |
| 9 | `kubectl apply -f deployment.yaml` sends the manifest to the **API server**. |
| 10 | Control plane works out the desired state (details below). |
| 11 | **kubelet** on the node sees a pod assigned to its node. |
| 12 | kubelet tells the **container runtime** (containerd) to run the container. |
| 13 | Container runtime **pulls the image** from the registry. |
| 14 | Container starts inside the **Pod**. |
| 15 | **Users** reach the app through a **Service**. |

---

## Control plane components (step 10)

| Component | Role |
|-----------|------|
| **kube-apiserver** | Front door of the cluster. Every component and `kubectl` talks only to it. |
| **etcd** | Key-value store holding the whole cluster state. **Only the API server talks to etcd.** |
| **controller-manager** | Runs controllers that keep actual state = desired state (Deployment → ReplicaSet → Pods). |
| **scheduler** | Picks which node each new pod should run on (based on CPU, memory, rules). |
| **cloud-controller-manager** | Talks to the cloud provider (e.g. creates a GCP load balancer for a `LoadBalancer` Service). |

## Node components

| Component | Role |
|-----------|------|
| **kubelet** | Agent on every node. Makes sure the pods assigned to the node are running and healthy. |
| **container runtime** | Actually runs containers (containerd on GKE). |
| **kube-proxy** | Sets up networking rules so Service traffic reaches the right pods. |

---

## Simple multi-stage Dockerfile (.NET)

The app is built inside Docker itself, in two stages:

```dockerfile
# Stage 1: build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: runtime (small image, no SDK)
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

- **Stage 1** uses the big SDK image to compile and publish the app.
- **Stage 2** starts fresh from the small runtime image and copies only the published output.
- Result: the final image has no SDK and no source code, only what's needed to run.

---

## Quick commands

```bash
# Build and push image
docker build -t <registry>/<app>:<tag> .
docker push <registry>/<app>:<tag>

# Deploy
kubectl apply -f deployment.yaml

# Check
kubectl get deployments
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get svc
```

---

## One-line summary

> CI turns code into an image in a registry. Kubernetes reads the manifest, stores it in etcd, schedules a pod onto a node, and the kubelet pulls the image and runs it; a Service exposes it to users.
