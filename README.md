# Kubernetes (k8s)
- meaning in Greek helmsman or pilot 
- open source system for automating deployment, scaling and management of containerized applications.
- portable, extensible, open source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation.
- builds upon 15 years of experience of running production workloads at Google. Open sourced in 2014.
- gives freedom to take advantage of on-premises, hybrid, or public cloud infrastructure to move your workloads

## Kubernetes Components

![Kubernetes Components](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

- Cluster is set of worker machines called **Node**, which run containerized applications. Every cluster has atleast one **Worker Node**
  - **kubelet**
    - Agent that runs on each node that makes sure containers are running in pod. Kubeletes ensures containers described in PodSpecs are runnin and healthy.
  - **kube-proxy**
    - Implements part of kubernetes service concept.
    - Maintains n/w rules on nodes, which allow n/w communication to your pods from n/w sessions inside or outside of your cluster.
    - Uses OS packet filtering layer if availanle else forwards the traffic itself.
  - **container runtime**
    - A software responsible for running containers.
    - Can support any which implements standard CRI like containerd, CRI-O. 
    - Docker is actually not a container runtime as it doesn't implement CRI, however it has set of tools that sits on top of containrd. As an exception, Kubernetes had provided an explicit support via Docker shim, but is now stopped supporting docker since v1.20.
  - **Addons**
    - DNS - DNS server that serves DNS records kubernetes services.
    - WEB UI (Dashboard) - UI to manage and troubleshoot cluster and apps running in cluster.
    - Container Resource Monitoring- Records generic  time series metrics of containers in central database and provided UI for browsing that data.
    - Cluster-Level logging - Mechanism to save container logs to central storage for search/browsing
    - CNI implementors like Calico, Flannel, Weave etc.

- **Control Plane** is container orchestration layer that exposes API and interfaces to define, deploy and manage lifecycle of containers. Scheduling, detecting and scheduling of cluster events.
  - **kube-apiserver**
    - API Server exposes kubernetes API that acts as frontent for control plane.
    - scales horizontally.
    - Employs OpenAPI v2 specs accessible using REST over http. kebectl internally uses http requests only. 
  - **kube-scheduler**
    - watches for newly created pods with unassigned node and selects a node for them to run on.
    - Governing factors: individual and collective resource requirements, hw/sw/policy constraints, affinity/anti-affinity specs, data locality, inter workload interference and deadlines.
  - **kube-controller-manager**
    - Runs controller processes, though logically seperate, they are compiled into one binary and runs as single process.
      - **Node Controller**: detects and responds to node going down.
      - **Job Controller**
      - **Endpoint Controller**: joins services to pods using endpoints.
      - **Service Account and Token Controller**: default accounts and API access tokens for new namespaces.
  - **cloud-controller-manager**:
  - **etcd**
    - Stores all cluster data.

## Kubernetes Objects
  - Persistent entities in k8s system, those represnt state of your cluster
    - what containerized applications are running
    - resources available to those apps
    - policies around apps' behaviour - restart, upgrade, fault tolerance etc.
  - With objects one describe the desired state (called **spec**), which k8s constantly work to reach that state in series of iterations (called **status**)
  - required fields: **apiVersion**, **kind**, **metadata** *(name + UID + optional namespace, and more)*, **spec**
  - k8s updated field: **status**
  - we write these objects in yaml which kubectl converts inot json to send it to API server

## Traditional Deployment:
- Physical servers
- No resource boundaries for applications leading to resource allocation issues
- Expensive to maintain multiple physical servers to run individual applications.
- Not scaled well as resources were under utilized.

## Virtual Deployment:
- Multiple VMs running on single maching while sharing the CPU and Memory
- Each VM is a full machine running all components including its own OS on top of virtualized hardware.
- Better resource utlization and better scalability as app can be added or updated easily, reduces hardware cost.
- Provides a level of security as info of one application cannot be freely accessed by another application

## Container Deployment
- Unlike VM, containers have relaxed isolation properties to share OS among applications.
- As they are decoupled from underlying infrastructure, they are portable across clouds and OS distributions.

- Service Discovery and Load balancing
  - Exposes a container using DNS name or using their own IP address
- Storage Orchestration
  - Allows to mount automatically a storage system of your choice - local, public cloud etc.
- Automated rollouts and rollbacks
  - Changes actual state to desired state that you described at a controlled rate.
- Automatic bin packing
  - Fits containers onto provided worker nodes to make best use of resources as you configured.
- Self healing
  - Restarts failed containers, replaces, kills containers that don't respond to user defined health check and doesn't advertise them until ready to serve.
- Secret and configuration management
  - You can deploy and update secrets (passwords, tokens) and application config without rebuilding container images and without exposing secrets in your stack configuration.

- Not a PaaS system: as it operates at container level than at hardware level, though having generally applicable features like deployment, scaling, load balancing and allows integration of logging, monitoring and alerting in pluggable manner.
- Does not build your application or deploy source code.
- Does not provide application level services like queue, db, caches, storage.
- Does not dictate logging, monitoring and alerting solution of its own, but provides integration mechanism.


# Architecture

- Control Plane
- Node 
  - Node Conditions: Kontrol plane allocates TAINTS to nodes corresponding to node conditions, which scheduler takes into consideration when assigning pod to node. Pods can have TOLERATIONS that let them run on node even though it has any specific taint.
    - Ready
    - DiskPressure
    - MemoryPressure
    - PIDPressure
    - NetworkUnavailable
  - 
  - kubelet
  - container runtime like docker* (deprecated in v1.20), containerd, CRI-O
  - kube-proxy

## Containers
  - images: a ready to run software package containing everything needed to run intended application - code, runtime, application, system libraries, default values for any essential settings. By default they are immutable i.e. you cannot change code of already running container, which requires rebuilding image.
    - Should avoid using :latest tag as makes it harder to track and more difficult to rollback.
    - image pull policy: created only once when object is first created and not updated if tag later changes.
      - **IfNotPresent** (default if image with tag provided)
      - **Always** (efficient with reliable registry, default if tag is :latest or no tag) - queries registry to resolve image name to an image digest. Uses cached images if image with that digest exists locally.
      - **Never** - only local or pre-pulled images else fails to start container
    - ImagePullBackoff container could not start as kubernetes could not pull image and it will try pulling image until backoff delay reaches to 5 mins.
    - Accessing private registry using ImagePullSecrets (recommended)
      - `kubectl create secret docker-registry <myregistrykey> --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL`
      - Add imagePullSecrets to each pod
        ```
        kind: Pod
        spec:
          containers:
          - name: foo
            image: janedoe/awesomeapp:v1
          imagePullSecrets:
          - name: myregistrykey
        ```
      - or to service account `default`:
          `kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'`


## Workloads

### Deployment
Declarative updates for Pods and ReplicaSets. Describe desired state in a Deployment and Deployment controller changes actual state to desired state at controlled rate.

- To rollout a ReplicSet
- To declare new state of Pods by updating PoTemplateSpec
- To rollback to an earlier Deployment revision
- To scale up Deployment to facillitate more load
- To pause the rollout of a Deployment
- To use status of Deployment
- To clean up older ReplicaSets

> **Note**: You must specify selector and pod template labels. **Do not overlap** labels or selectors with other controllers, though it is technically allowed, may lead to conflict and unexpected behaviour.
  
`kubectl rollout status deployment/nginx-deployment`
> Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out

- Name of deployment:   nginx-deployment
- Name of ReplicaSet:   nginx-deployment-\<pod-template-hash\>
- Name of Pod replicas: 
  - nginx-deployment-\<pod-template-hash\>-7ci7o
  - nginx-deployment-\<pod-template-hash\>-kzszj
  - nginx-deployment-\<pod-template-hash\>-qqcnn 

pod-template-hash label is added by Deployment controller to every ReplicaSet to ensure they do not overlap.

When you update a deployment, it creates new ReplicaSet and scale it up to 1, then scale down the old ReplicaSet by 1. So it ensures at minimum, expected  number of replicas (say 3 in above example) and maximum plus one (i.e. 4 in above example) are running at any time during update.

> replicas - maxUnavailable \< availableReplicas \< replicas + maxSurge

As a result, there could be more Pods than expected during a rollout, and that the total resources consumed by the Deployment is more than replicas + maxSurge until the terminationGracePeriodSeconds of the terminating Pods expires.

#### Deployment Strategies

By default, the maximum number of Pods that can be unavailable during the update and the maximum number of new Pods that can be created, is one. Both options can be configured to either numbers or percentages (of Pods).

In Kubernetes, updates are versioned and any Deployment update can be reverted to a previous (stable) version.

1. Rolling Deployment
    - Default strategy which replaces existing pods with new version one by one
    - Checks if new pod is ready using readyness probe before scale down old pods.
    - No downtime
      ```
      spec:
        strategy:
          rollingUpdate: 
            maxSurge: <N or %>        # max pods allowed to create at one time (number or percentage)
            maxUnavailable: <N or %>  # max pods allowed to be unavailable during rollout (number or percentage)
      ```
2. Recreate
   - Terminates all pods before replacing with new version
      ```
      spec:
        type: Recreate
      ```
3. Ramped slow rollout
    - Rolls out replicas of the new version, while in parallel, shutting down old replicas
    - Difference with default RollingOut is you control pace at which new replicas are rolled out
    - One can choose number of replicas to roll out each time but ensure no pods become unavailable
    - Reduces risk of an update but takes long time to roll out
      ```
      spec:
        strategy:
          rollingUpdate:
            maxSurge: 1
            maxUnavailable: 0     # Always 0
      ```
4. Best-effort controlled rollout
    - Enables faster rollout at the expense of certain percentage of downtime among your nodes
      ```
      spec:
        strategy:
          rollingUpdate:
            maxSurge: %           # Percentage of tolerance of downtime
            maxUnavailable: 0     # Always 0
      ```
5. Blue/Green mirroring
    - Both old and new versions are running simulaneously
    - While only old version (blue) is available to users, new one is with QA on a separate service or port forwarding
    - After new verson is tested, service is switched to new version while old version is scaling down.
    - This is achieved with Service spec selector selecting to new version post testing.
      ```
      kind: Service
      metadata:
        name: awesomeapp
      spec:
        selector:
          app: awesomeapp
          version: "02"
      ```
6. Canary releases
    - Release a new version to a subset of users, then proceed to a full rollout
    - Similar to Blue/Green deployment, but more controlled and use a more **progressive delivery** phased-in approach.
    - While it can be done with k8s resources, it's convenient to use service mesh like **Istio** or **Linkerd** and can be automated with **Argo Rollouts** or more advanced **Flagger (From WeaveWorks)**
    
      **Dark Deployment or A/B testing**
      - The difference between a dark deployment and a canary is that dark deployments deal with features in the front-end rather than the backend as is the case with canaries.

### StatefulSets
- Manages stateful applications
- It's like Deployment, but provides guarantees about the ordering and uniqueness of the deployed Pods
- Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec.
- Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.
- StatefulSet is ideal when you are to attach persistent volume to your pods. Although individual Pods in a StatefulSet are susceptible to failure, the persistent Pod identifiers make it easier to match existing volumes to the new Pods that replace any that have failed.
- StatefulSets require a Headless Service to be responsible for network identity of Pods. 

1. Stable, unique network identifiers.
2. Stable, persistent storage.
3. Ordered, graceful deployment and scaling.
4. Ordered, automated rolling updates.



## Services, Load Balancing and Networking

## Storage
- Persistent Volume - a piece of storage in cluster provisioned by admin or dynamically provisioned using storage classes. 
- Have lifecycle independent of any individual pod that uses it.
- Provides an API that abstracts details of how storage is provisioned from how it is consumed.
  - Provisioning
    - Static: cluster administer creates real storage volumes.
    - Dynamic: usage default storage class
  - Types:
    - HostPath: uses a file or directory on the Node to emulate network-attached storage. Useful in development.
- Persistent Volume Claim - a request for storage by a user. PVCs consume PV resources, with request of specific size and access modes (RWOnce, ROMany, RWMany)
## Configuration

## Security

## Policies

## Scheduling, Preemption and Eviction
