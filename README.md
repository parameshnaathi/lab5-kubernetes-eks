# Kubernetes on Amazon EKS - Managed Node Group

## Objective

This lab builds a standard Amazon EKS cluster with a **2-node EC2 managed node group** and then deploys a Kubernetes application using the manifests stored in this repository.

This version intentionally uses:

```text
EKS Auto Mode       = Disabled
Worker model        = EKS Managed Node Group
Worker OS           = Amazon Linux 2023
Desired nodes       = 2
Minimum nodes       = 2
Maximum nodes       = 3
```

The lab covers:

- EKS cluster IAM role
- EKS node IAM role
- EKS control plane
- VPC and subnet selection
- Managed node group
- Two EC2 worker nodes
- EKS add-ons
- kubeconfig and kubectl access
- Namespace
- ConfigMap
- Secret
- Deployment
- ReplicaSet
- Pods
- Service
- Labels and selectors
- Requests and limits
- Readiness and liveness probes
- Self-healing
- Scaling
- Rolling updates
- Rollback
- Logs, events, and basic troubleshooting
- Cleanup

The Kubernetes YAML definitions are maintained under `manifests/`. Refer to those files directly rather than duplicating the full manifests in this README.

---

# Lab Architecture

```text
                              AWS
                               |
                               v
                     Amazon EKS Control Plane
                  +-----------------------------+
                  | API Server                  |
                  | Scheduler                   |
                  | Controller Manager          |
                  | etcd                        |
                  +--------------+--------------+
                                 |
                                 v
                      EKS Managed Node Group
                                 |
                    +------------+------------+
                    |                         |
                    v                         v
              EC2 Worker Node 1         EC2 Worker Node 2
              Amazon Linux 2023         Amazon Linux 2023
              Public Subnet A           Public Subnet B
                    |                         |
                    +------------+------------+
                                 |
                              Pods
                         +-------+-------+
                         |               |
                         v               v
                       Pod-1           Pod-2
                         \               /
                          \             /
                           +-----------+
                                |
                             Service
                            ClusterIP
```

For this lab, the cluster can use the existing default VPC:

```text
VPC:
172.31.0.0/16
```

with two subnets in different Availability Zones.

For production environments, worker nodes are commonly placed in private subnets. This lab uses the simpler existing-network setup so the focus remains on Kubernetes.

---

# AWS Resources Created

The complete EKS setup contains more than just the Kubernetes cluster.

```text
VPC
 |
 +-- Subnet A
 |
 +-- Subnet B
 |
 +-- EKS Cluster Security Group
 |
 +-- EKS Control Plane
 |      |
 |      +-- Cluster IAM Role
 |
 +-- Managed Node Group
        |
        +-- Node IAM Role
        |
        +-- Auto Scaling Group
        |
        +-- EC2 Node 1
        |
        +-- EC2 Node 2
        |
        +-- EKS Optimized AL2023 AMI

Kubernetes Add-ons
 |
 +-- Amazon VPC CNI
 +-- CoreDNS
 +-- kube-proxy
```

The important relationship is:

```text
Cluster IAM Role
      |
      v
EKS Control Plane


Node IAM Role
      |
      v
Managed Node Group
      |
      v
EC2 Worker Nodes
```

---

# Repository Structure

```text
lab5-kubernetes-eks/
│
├── manifests/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── namespace.yaml
│   ├── secret.yaml
│   └── service.yaml
│
└── README.md
```

Manifest responsibility:

```text
namespace.yaml
    =
Logical application namespace

configmap.yaml
    =
Non-sensitive application configuration

secret.yaml
    =
Demo sensitive application value

deployment.yaml
    =
Application Pods, replicas, probes, resources,
ConfigMap and Secret consumption

service.yaml
    =
Stable internal access to matching Pods
```

---

# Quick Command Reference

| Command | Purpose | Example |
|---|---|---|
| `aws sts get-caller-identity` | Verify AWS identity | `aws sts get-caller-identity` |
| `aws eks list-clusters` | List EKS clusters | `aws eks list-clusters` |
| `aws eks update-kubeconfig` | Configure kubectl | `aws eks update-kubeconfig --region us-east-1 --name arrise-lab9-eks` |
| `kubectl get nodes` | Check worker nodes | `kubectl get nodes` |
| `kubectl get nodes -o wide` | Detailed node information | `kubectl get nodes -o wide` |
| `kubectl get pods -A` | List all Pods | `kubectl get pods -A` |
| `kubectl get pods` | List application Pods | `kubectl get pods` |
| `kubectl get pods -o wide` | Pod IP and node placement | `kubectl get pods -o wide` |
| `kubectl get deployment` | Check Deployments | `kubectl get deployment` |
| `kubectl get replicasets` | Check ReplicaSets | `kubectl get replicasets` |
| `kubectl get svc` | Check Services | `kubectl get svc` |
| `kubectl get endpointslice` | Check Service backends | `kubectl get endpointslice` |
| `kubectl describe pod <pod>` | Detailed Pod troubleshooting | `kubectl describe pod <pod>` |
| `kubectl logs <pod>` | View logs | `kubectl logs <pod>` |
| `kubectl get events` | View events | `kubectl get events --sort-by=.lastTimestamp` |
| `kubectl scale deployment` | Scale Pods | `kubectl scale deployment arrise-web --replicas=4` |
| `kubectl rollout status` | Check rollout | `kubectl rollout status deployment/arrise-web` |
| `kubectl rollout history` | View rollout history | `kubectl rollout history deployment/arrise-web` |
| `kubectl rollout undo` | Roll back | `kubectl rollout undo deployment/arrise-web` |

---

# Part 1 - Verify AWS Access

From PowerShell:

```powershell
aws sts get-caller-identity
```

Confirm that the expected AWS account and IAM principal are returned.

Also verify the AWS Region you plan to use.

Example:

```powershell
aws configure get region
```

---

# Part 2 - Select the VPC and Two Subnets

For this lab, use:

```text
VPC:
172.31.0.0/16
```

Select two subnets in different Availability Zones.

You can verify them with:

```powershell
aws ec2 describe-subnets `
  --filters "Name=vpc-id,Values=<VPC-ID>" `
  --query "Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}" `
  --output table
```

Example design:

```text
VPC
172.31.0.0/16
      |
      +---------------------------+
      |                           |
      v                           v
Public Subnet A              Public Subnet B
AZ-A                         AZ-B
      |                           |
      +-------------+-------------+
                    |
                    v
                   EKS
```

Using two Availability Zones avoids putting all worker capacity in one AZ.

---

# Part 3 - Create the EKS Cluster IAM Role

The EKS control plane requires an IAM role.

Go to:

```text
AWS Console
    |
    v
IAM
    |
    v
Roles
    |
    v
Create role
```

Choose:

```text
Trusted entity:
AWS service

Use case:
EKS

Role:
EKS - Cluster
```

Example role name:

```text
arrise-lab9-eks-cluster-role
```

Ensure the role has:

```text
AmazonEKSClusterPolicy
```

Relationship:

```text
arrise-lab9-eks-cluster-role
            |
            v
       EKS Control Plane
```

The node role created later must be a separate IAM role.

---

# Part 4 - Create the EKS Cluster

Open:

```text
AWS Console
    |
    v
Amazon EKS
    |
    v
Clusters
    |
    v
Create cluster
```

Choose:

```text
Custom configuration
```

Important:

```text
EKS Auto Mode:
OFF
```

This lab intentionally uses a traditional managed node group.

Configure:

```text
Cluster name:
arrise-lab9-eks

Cluster IAM role:
arrise-lab9-eks-cluster-role

Kubernetes version:
Use a currently supported standard-support version
```

Networking:

```text
VPC:
172.31.0.0/16 VPC

Subnets:
Select two subnets in different AZs
```

For this lab, use the existing public subnets.

Keep the default EKS cluster security group unless you specifically need a custom design.

Create the cluster.

Wait until:

```text
Cluster status:
ACTIVE
```

before creating nodes.

---

# Part 5 - EKS Control Plane Components

Amazon EKS manages the Kubernetes control plane.

Conceptually:

```text
EKS Control Plane
      |
      +-- API Server
      |
      +-- Scheduler
      |
      +-- Controller Manager
      |
      +-- etcd
```

You do not SSH into or directly manage these control-plane hosts.

AWS operates them for you.

Worker nodes communicate with the Kubernetes API Server and run your application workloads.

---

# Part 6 - Create the Node IAM Role

The EC2 worker nodes require their own IAM role.

Go to:

```text
IAM
 |
Roles
 |
Create role
```

Choose:

```text
Trusted entity:
AWS service

Use case:
EC2
```

Example role name:

```text
arrise-lab9-node-role
```

For this lab, ensure the role has the permissions required by EKS worker nodes.

Common AWS managed policies used for a standard lab setup include:

```text
AmazonEKSWorkerNodePolicy
AmazonEC2ContainerRegistryPullOnly
```

The Amazon VPC CNI also needs AWS permissions. For a simple lab, follow the EKS console's recommended add-on/IAM configuration. In production, the CNI permissions should preferably be assigned separately to the networking add-on rather than unnecessarily expanding the node role.

Important:

```text
Cluster IAM Role
        !=
Node IAM Role
```

Relationship:

```text
Node IAM Role
      |
      v
EC2 Worker Node
      |
      +-- kubelet
      |
      +-- containerd
      |
      +-- AWS API permissions
```

---

# Part 7 - Create the Managed Node Group

After the EKS cluster becomes:

```text
ACTIVE
```

go to:

```text
Amazon EKS
    |
    v
arrise-lab9-eks
    |
    v
Compute
    |
    v
Node groups
    |
    v
Add node group
```

Configure:

```text
Node group name:
arrise-lab9-ng

Node IAM role:
arrise-lab9-node-role
```

Choose:

```text
AMI:
Amazon Linux 2023 x86_64 Standard

Capacity type:
On-Demand
```

For a small lab, use an instance type such as:

```text
t3.medium
```

Scaling:

```text
Desired size:
2

Minimum size:
2

Maximum size:
3
```

Networking:

```text
Subnet A
Subnet B
```

Use the same two subnets selected for the cluster.

For this lab, SSH access to the nodes is not required.

Create the node group.

---

# Part 8 - What EKS Creates for the Node Group

A managed node group creates and manages AWS infrastructure for the worker nodes.

Conceptually:

```text
EKS Managed Node Group
        |
        v
EC2 Auto Scaling Group
        |
        +-----------------------+
        |                       |
        v                       v
EC2 Worker Node 1         EC2 Worker Node 2
Amazon Linux 2023         Amazon Linux 2023
```

The nodes use an EKS-optimized Amazon Linux 2023 AMI.

The AMI already contains the components necessary to join the EKS cluster, including the Kubernetes node software and container runtime.

With:

```text
desired = 2
minimum = 2
maximum = 3
```

the initial expected state is:

```text
2 worker nodes
```

---

# Part 9 - Wait for the Node Group

Wait until:

```text
Node group status:
ACTIVE
```

Then configure kubectl.

From PowerShell:

```powershell
aws eks update-kubeconfig `
  --region <REGION> `
  --name arrise-lab9-eks
```

Example:

```powershell
aws eks update-kubeconfig `
  --region us-east-1 `
  --name arrise-lab9-eks
```

Check:

```powershell
kubectl config current-context
```

---

# Part 10 - Verify the Two Worker Nodes

Run:

```powershell
kubectl get nodes
```

Expected conceptually:

```text
NAME              STATUS   ROLES    VERSION
ip-xxxxx          Ready    <none>   ...
ip-yyyyy          Ready    <none>   ...
```

You should see:

```text
2 Ready nodes
```

Detailed view:

```powershell
kubectl get nodes -o wide
```

Check:

```text
INTERNAL-IP
OS-IMAGE
KERNEL-VERSION
CONTAINER-RUNTIME
```

For this lab the OS should be Amazon Linux 2023 rather than Bottlerocket.

---

# Part 11 - Verify EKS Add-ons

Run:

```powershell
kubectl get pods -n kube-system
```

In a standard EKS managed-node setup you normally see Kubernetes infrastructure components such as:

```text
aws-node
coredns
kube-proxy
```

Depending on the cluster configuration, you may also have:

```text
metrics-server
```

Relationship:

```text
aws-node
    =
Amazon VPC CNI networking


coredns
    =
Kubernetes DNS


kube-proxy
    =
Service networking on nodes
```

Check add-ons in AWS:

```text
EKS
 |
arrise-lab9-eks
 |
Add-ons
```

---

# Part 12 - Understand Node Components

Each Linux worker node runs components such as:

```text
EC2 Worker Node
      |
      +-- kubelet
      |
      +-- containerd
      |
      +-- kube-proxy
      |
      +-- VPC CNI networking
      |
      +-- Application Pods
```

The simplified workload flow is:

```text
kubectl
   |
   v
API Server
   |
   v
Scheduler
   |
   v
Select worker node
   |
   v
kubelet
   |
   v
containerd
   |
   v
Pod starts
```

---

# Part 13 - Create the Namespace

Refer to:

```text
manifests/namespace.yaml
```

Apply:

```powershell
kubectl apply -f manifests/namespace.yaml
```

Verify:

```powershell
kubectl get namespaces
```

Set the namespace:

```powershell
kubectl config set-context --current --namespace=arrise-dev
```

Verify:

```powershell
kubectl config view --minify | Select-String namespace
```

---

# Part 14 - Create the ConfigMap

Refer to:

```text
manifests/configmap.yaml
```

Apply:

```powershell
kubectl apply -f manifests/configmap.yaml
```

Verify:

```powershell
kubectl get configmap
```

Purpose:

```text
ConfigMap
    =
Non-sensitive application configuration
```

In this lab:

```text
ConfigMap
    |
    | index.html
    v
Deployment
    |
    v
nginx Pod
```

---

# Part 15 - Create the Secret

Refer to:

```text
manifests/secret.yaml
```

Apply:

```powershell
kubectl apply -f manifests/secret.yaml
```

Verify:

```powershell
kubectl get secrets
```

Purpose:

```text
Secret
    =
Sensitive application configuration
```

Relationship:

```text
Secret
   |
   | APP_TOKEN
   v
Deployment
   |
   v
Pod environment
```

The lab value is intentionally fake.

Do not commit real production secrets into the repository.

---

# Part 16 - Deploy the Application

Refer to:

```text
manifests/deployment.yaml
```

Apply:

```powershell
kubectl apply -f manifests/deployment.yaml
```

Watch:

```powershell
kubectl get pods -w
```

Wait for:

```text
1/1 Running
```

for both replicas.

Stop watching with:

```text
Ctrl+C
```

---

# Part 17 - Deployment Relationship

```text
Deployment
    |
    v
ReplicaSet
    |
    +----------+
    |          |
    v          v
  Pod-1      Pod-2
```

The Deployment defines:

```text
Desired replicas
Container image
Labels
ConfigMap mount
Secret reference
Readiness probe
Liveness probe
CPU request
Memory request
CPU limit
Memory limit
Rolling update strategy
```

---

# Part 18 - Verify Pod Placement Across Nodes

Run:

```powershell
kubectl get pods -o wide
```

Inspect:

```text
NAME
IP
NODE
```

You may see:

```text
Pod-1 -> Worker Node 1
Pod-2 -> Worker Node 2
```

However, Kubernetes does not guarantee one Pod per node by default.

The Scheduler can place both Pods on one node if that placement satisfies all scheduling rules and resource requirements.

If strict spreading is required, Kubernetes affinity or topology-spread rules should be configured.

---

# Part 19 - Labels and Selectors

Check:

```powershell
kubectl get pods --show-labels
```

The Pods use a label such as:

```text
app=arrise-web
```

Relationship:

```text
Deployment selector
       |
       v
app=arrise-web
       |
       v
Pods
```

The Service uses the same label to discover its backend Pods.

---

# Part 20 - Requests and Limits

Refer to:

```text
manifests/deployment.yaml
```

Memory shortcut:

```text
REQUEST
    =
Resource requirement used during scheduling


LIMIT
    =
Maximum resource boundary for a container
```

The Scheduler considers requests when choosing between:

```text
Worker Node 1
Worker Node 2
```

If neither node can satisfy the request:

```text
Pod
 |
 v
Pending
```

Troubleshoot with:

```powershell
kubectl describe pod <pod-name>
```

---

# Part 21 - Readiness and Liveness

Refer to the probes in:

```text
manifests/deployment.yaml
```

Remember:

```text
READINESS
    =
Should the Service send traffic to this Pod?


LIVENESS
    =
Should kubelet restart this container?
```

Readiness failure:

```text
Pod
 |
Not Ready
 |
Removed from Service endpoints
```

Liveness failure:

```text
Container unhealthy
 |
kubelet
 |
Restart container
```

---

# Part 22 - Create the Service

Refer to:

```text
manifests/service.yaml
```

Apply:

```powershell
kubectl apply -f manifests/service.yaml
```

Verify:

```powershell
kubectl get svc
```

The lab uses:

```text
ClusterIP
```

which provides stable internal access.

---

# Part 23 - ConfigMap, Secret, Deployment and Service

The complete relationship is:

```text
             ConfigMap
                 |
                 | non-sensitive config
                 v
              Deployment
                 ^
                 |
                 | sensitive value
                 |
               Secret
                 |
                 v
              ReplicaSet
                 |
          +------+------+
          |             |
          v             v
        Pod-1         Pod-2
          |             |
          +------+------+
                 ^
                 |
          app=arrise-web
                 |
              Service
                 |
                 v
           Stable endpoint
```

Simple meaning:

```text
ConfigMap
    =
Application configuration

Secret
    =
Sensitive configuration

Deployment
    =
Run and maintain the application

Service
    =
Reach the application
```

---

# Part 24 - Check Service Endpoints

Run:

```powershell
kubectl get endpointslice
```

Compare with:

```powershell
kubectl get pods -o wide
```

Flow:

```text
Service
   |
   v
EndpointSlice
   |
   +-------------+
   |             |
   v             v
Pod IP 1       Pod IP 2
```

---

# Part 25 - Test the Application

Run:

```powershell
kubectl run test-client `
  --image=busybox:1.36 `
  --restart=Never `
  --rm -it `
  -- wget -qO- http://arrise-web
```

The request uses:

```text
http://arrise-web
```

instead of a Pod IP.

This demonstrates Kubernetes DNS and Service discovery.

---

# Part 26 - Verify ConfigMap and Secret

Get the Pods:

```powershell
kubectl get pods
```

Enter one:

```powershell
kubectl exec -it <pod-name> -- /bin/sh
```

Check Secret:

```sh
echo $APP_TOKEN
```

Check ConfigMap-mounted content:

```sh
cat /usr/share/nginx/html/index.html
```

Exit:

```sh
exit
```

---

# Part 27 - Self-Healing

List Pods:

```powershell
kubectl get pods
```

Delete one:

```powershell
kubectl delete pod <pod-name>
```

Watch:

```powershell
kubectl get pods -w
```

Flow:

```text
Desired replicas = 2
        |
Pod deleted
        |
Actual replicas = 1
        |
ReplicaSet detects mismatch
        |
Replacement Pod created
        |
Actual replicas = 2
```

This is Kubernetes reconciliation.

---

# Part 28 - Scale the Application

Scale:

```powershell
kubectl scale deployment arrise-web --replicas=4
```

Check:

```powershell
kubectl get pods -o wide
```

The Scheduler distributes Pods according to:

```text
Available CPU
Available memory
Scheduling constraints
Taints/tolerations
Affinity rules
Topology rules
```

Reset:

```powershell
kubectl scale deployment arrise-web --replicas=2
```

---

# Part 29 - Pod Scaling vs Node Scaling

These are separate concepts.

```text
Pod Scaling
    =
Change Deployment replicas


Node Scaling
    =
Change number of EC2 workers
```

Example:

```text
Deployment
2 Pods -> 4 Pods
        |
        v
Scheduler checks existing nodes
        |
        +-- Enough capacity
        |       |
        |       v
        |  Schedule Pods
        |
        +-- Not enough capacity
                |
                v
       Additional node capacity required
```

The node group in this lab is configured as:

```text
Minimum:
2

Desired:
2

Maximum:
3
```

Changing Pod replicas does not by itself guarantee that the managed node group automatically increases to 3. Automatic node scaling requires an autoscaling solution/configuration.

---

# Part 30 - Rolling Update

Run:

```powershell
kubectl set env deployment/arrise-web APP_VERSION=v2
```

Watch:

```powershell
kubectl get pods -w
```

Then:

```powershell
kubectl rollout status deployment/arrise-web
```

Flow:

```text
Existing ReplicaSet
       |
       v
New Deployment revision
       |
       v
New ReplicaSet
       |
       v
New Pods become Ready
       |
       v
Old Pods terminate gradually
```

---

# Part 31 - Rollout History and Rollback

History:

```powershell
kubectl rollout history deployment/arrise-web
```

Rollback:

```powershell
kubectl rollout undo deployment/arrise-web
```

Verify:

```powershell
kubectl rollout status deployment/arrise-web
```

---

# Part 32 - Pod Logs

```powershell
kubectl logs <pod-name>
```

Follow:

```powershell
kubectl logs -f <pod-name>
```

For multiple containers:

```powershell
kubectl logs <pod-name> -c <container-name>
```

---

# Part 33 - Describe and Events

Describe:

```powershell
kubectl describe pod <pod-name>
```

Inspect:

```text
Node
Pod IP
Container status
Requests
Limits
Readiness
Liveness
Volumes
Events
```

Cluster events:

```powershell
kubectl get events --sort-by=.lastTimestamp
```

Common events:

```text
Scheduled
Pulling
Pulled
Created
Started
Unhealthy
FailedScheduling
FailedMount
BackOff
Killing
```

---

# Part 34 - Basic Kubernetes Troubleshooting Flow

Use this order:

```text
kubectl get pods
        |
        v
Pod status
        |
        v
kubectl describe pod
        |
        v
Events
        |
        v
kubectl logs
        |
        v
Deployment
        |
        v
Service
        |
        v
Labels / selectors
        |
        v
EndpointSlice
        |
        v
ConfigMap / Secret
        |
        v
Requests / limits
        |
        v
Nodes
```

Useful commands:

```powershell
kubectl get nodes -o wide
kubectl get pods -o wide
kubectl get deployment
kubectl get replicasets
kubectl get svc
kubectl get endpointslice
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events --sort-by=.lastTimestamp
```

---

# Part 35 - Apply All Application Manifests

After reviewing the files:

```powershell
kubectl apply -f manifests/
```

Verify:

```powershell
kubectl get all
kubectl get configmap
kubectl get secrets
```

---

# Part 36 - Complete Kubernetes Application Flow

```text
                     Namespace
                    arrise-dev
                         |
          +--------------+--------------+
          |                             |
          v                             v
      ConfigMap                       Secret
          |                             |
          +--------------+--------------+
                         |
                         v
                    Deployment
                         |
                         v
                    ReplicaSet
                         |
                  +------+------+
                  |             |
                  v             v
                Pod-1         Pod-2
                  |             |
                  +------+------+
                         ^
                         |
                  app=arrise-web
                         |
                      Service
                         |
                         v
                   ClusterIP DNS
                         |
                         v
                   http://arrise-web
```

---

# Part 37 - Complete EKS Flow

```text
AWS Account
    |
    v
VPC + 2 Subnets
    |
    v
EKS Cluster IAM Role
    |
    v
EKS Control Plane
    |
    v
Managed Node Group
    |
    +-- Node IAM Role
    |
    +-- Auto Scaling Group
    |
    +----------+----------+
    |                     |
    v                     v
AL2023 Node 1         AL2023 Node 2
    |                     |
    +----------+----------+
               |
               v
          Kubernetes Pods
               |
               v
             Service
```

---

# Part 38 - Cleanup Application Resources

Delete the application namespace:

```powershell
kubectl delete namespace arrise-dev
```

This removes the namespaced lab resources:

```text
Deployment
ReplicaSet
Pods
Service
ConfigMap
Secret
```

---

# Part 39 - Cleanup EKS Resources

Amazon EKS and EC2 worker nodes are billable.

Delete in dependency order:

```text
Application resources
        |
        v
Managed Node Group
        |
        v
EKS Cluster
        |
        v
Lab-specific IAM roles if no longer required
```

In the AWS Console:

```text
EKS
 |
arrise-lab9-eks
 |
Compute
 |
Delete arrise-lab9-ng
```

Wait until the node group is deleted.

Then:

```text
EKS
 |
Clusters
 |
arrise-lab9-eks
 |
Delete
```

Only delete shared/default VPC resources if you intentionally created them for the lab. Do not delete an account's default VPC just to clean up this exercise.

---