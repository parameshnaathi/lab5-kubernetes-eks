# Lab 5.1 - Kubernetes Production Troubleshooting Scenarios

## Prerequisite

This lab reuses the Kubernetes environment created in **Lab 5**.

Before starting, make sure the Lab 5 application is available:

```text
Namespace:
arrise-dev

Deployment:
arrise-web

Service:
arrise-web

ConfigMap:
arrise-web-config

Secret:
arrise-web-secret
```

Use the YAML files and EKS environment already created in Lab 5.

This lab does not create a new EKS cluster, VPC, subnet, node group, or application architecture.

---

# Objective

The goal of this lab is to intentionally create common Kubernetes production failures and troubleshoot them using a repeatable approach.

The scenarios covered are:

1. `ImagePullBackOff`
2. `CrashLoopBackOff`
3. `CreateContainerConfigError`
4. Readiness probe failure
5. Service selector mismatch
6. Pod stuck in `Pending`

---

# Quick Troubleshooting Commands

| Command | Purpose | Example |
|---|---|---|
| `kubectl get pods` | Check Pod status | `kubectl get pods` |
| `kubectl get pods -o wide` | Show Pod IP and node | `kubectl get pods -o wide` |
| `kubectl describe pod` | Check Pod details and events | `kubectl describe pod <pod>` |
| `kubectl logs` | Check current container logs | `kubectl logs <pod>` |
| `kubectl logs --previous` | Check previous crashed container logs | `kubectl logs <pod> --previous` |
| `kubectl get events` | View recent Kubernetes events | `kubectl get events --sort-by=.lastTimestamp` |
| `kubectl get deployment` | Check Deployment status | `kubectl get deployment` |
| `kubectl describe deployment` | Inspect Deployment | `kubectl describe deployment arrise-web` |
| `kubectl get svc` | Check Service | `kubectl get svc` |
| `kubectl get endpointslice` | Check Service backends | `kubectl get endpointslice` |
| `kubectl get pods --show-labels` | Check Pod labels | `kubectl get pods --show-labels` |
| `kubectl get secrets` | Check Secrets | `kubectl get secrets` |
| `kubectl get configmap` | Check ConfigMaps | `kubectl get configmap` |
| `kubectl get nodes --show-labels` | Check node labels | `kubectl get nodes --show-labels` |
| `kubectl rollout status` | Check rollout progress | `kubectl rollout status deployment/arrise-web` |
| `kubectl rollout undo` | Roll back failed Deployment | `kubectl rollout undo deployment/arrise-web` |
| `kubectl top nodes` | Check node resources | `kubectl top nodes` |
| `kubectl top pods` | Check Pod resources | `kubectl top pods` |

---

# Standard Troubleshooting Flow

Use the same order for most Kubernetes incidents:

```text
kubectl get
      |
      v
Identify symptom
      |
      v
kubectl describe
      |
      v
Check Events
      |
      v
kubectl logs
      |
      v
Check Deployment / ReplicaSet
      |
      v
Check Service / EndpointSlice
      |
      v
Check ConfigMap / Secret
      |
      v
Check node / resources
```

Do not immediately restart or delete resources before understanding the failure.

---

# Start With a Healthy Baseline

Set the Lab 5 namespace:

```powershell
kubectl config set-context --current --namespace=arrise-dev
```

Verify:

```powershell
kubectl get deployment
kubectl get pods
kubectl get svc
kubectl get endpointslice
```

Expected healthy state:

```text
Deployment:
READY 2/2

Pods:
1/1 Running

Service:
arrise-web

EndpointSlice:
Contains application Pod IPs
```

You can also reapply the known-good Lab 5 manifests if required:

```powershell
kubectl apply -f manifests/
```

---

# Scenario 1 - ImagePullBackOff

## Problem

The Pod cannot download the configured container image.

Common reasons:

```text
Wrong image name
Wrong image tag
Image does not exist
Private registry authentication problem
Registry/network connectivity issue
```

---

## Inject the Failure

Set an invalid nginx image:

```powershell
kubectl set image deployment/arrise-web `
  nginx=nginx:this-image-does-not-exist
```

Watch:

```powershell
kubectl get pods -w
```

Possible progression:

```text
Pending
   |
   v
ErrImagePull
   |
   v
ImagePullBackOff
```

---

## Troubleshoot

First:

```powershell
kubectl get pods
```

Find the failed Pod.

Then:

```powershell
kubectl describe pod <failed-pod>
```

Check the Events section.

You should see messages related to:

```text
Failed to pull image
ErrImagePull
ImagePullBackOff
```

Check the Deployment image:

```powershell
kubectl get deployment arrise-web `
  -o jsonpath="{.spec.template.spec.containers[*].image}"
```

Expected bad value:

```text
nginx:this-image-does-not-exist
```

---

## Root Cause Flow

```text
Deployment
    |
    v
Invalid image
    |
    v
Node attempts image pull
    |
    X
Image unavailable
    |
    v
ErrImagePull
    |
    v
ImagePullBackOff
```

---

## Recover

```powershell
kubectl rollout undo deployment/arrise-web
```

Verify:

```powershell
kubectl rollout status deployment/arrise-web
kubectl get pods
```

Expected:

```text
1/1 Running
```

---

# Scenario 2 - CrashLoopBackOff

## Problem

The container starts but repeatedly exits or crashes.

Common reasons:

```text
Application error
Bad command
Missing configuration
Application dependency unavailable
Probe failure
Resource problem
Invalid startup parameters
```

---

## Inject the Failure

Force the container to exit:

```powershell
kubectl patch deployment arrise-web `
  --type=strategic `
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","command":["/bin/sh","-c","echo simulated-application-crash; exit 1"]}]}}}}'
```

Watch:

```powershell
kubectl get pods -w
```

Possible progression:

```text
Running
   |
   v
Error
   |
   v
Restart
   |
   v
CrashLoopBackOff
```

---

## Troubleshoot

Check:

```powershell
kubectl get pods
```

Then:

```powershell
kubectl describe pod <failed-pod>
```

Look at:

```text
State
Last State
Reason
Exit Code
Restart Count
Events
```

Check current logs:

```powershell
kubectl logs <failed-pod>
```

Most importantly:

```powershell
kubectl logs <failed-pod> --previous
```

Expected:

```text
simulated-application-crash
```

---

## Why `--previous` Is Important

```text
kubectl logs
    =
Current container instance


kubectl logs --previous
    =
Previous crashed container instance
```

For `CrashLoopBackOff`, previous logs are often more useful because the container may have already restarted.

---

## Root Cause Flow

```text
Container starts
      |
      v
Application exits
      |
      v
Kubernetes restarts container
      |
      v
Application exits again
      |
      v
Backoff increases
      |
      v
CrashLoopBackOff
```

---

## Recover

```powershell
kubectl rollout undo deployment/arrise-web
```

Then:

```powershell
kubectl rollout status deployment/arrise-web
kubectl get pods
```

---

# Scenario 3 - CreateContainerConfigError

## Problem

Kubernetes cannot construct the container configuration.

A common cause is:

```text
Missing Secret
Missing ConfigMap
Wrong key
Wrong resource name
```

In this scenario, the Deployment will reference a Secret that does not exist.

---

## Inject the Failure

```powershell
kubectl patch deployment arrise-web `
  --type=strategic `
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","env":[{"name":"APP_TOKEN","valueFrom":{"secretKeyRef":{"name":"missing-secret","key":"APP_TOKEN"}}}]}]}}}}'
```

Check:

```powershell
kubectl get pods
```

The new Pod may show:

```text
CreateContainerConfigError
```

---

## Troubleshoot

Describe the failed Pod:

```powershell
kubectl describe pod <failed-pod>
```

Look at Events.

Expected message:

```text
secret "missing-secret" not found
```

Check available Secrets:

```powershell
kubectl get secrets
```

You should see:

```text
arrise-web-secret
```

but not:

```text
missing-secret
```

Inspect the Deployment:

```powershell
kubectl describe deployment arrise-web
```

---

## Root Cause Flow

```text
Deployment
    |
    v
References missing-secret
    |
    v
Pod creation
    |
    X
Secret not found
    |
    v
CreateContainerConfigError
```

---

## Recover

```powershell
kubectl rollout undo deployment/arrise-web
```

Verify:

```powershell
kubectl rollout status deployment/arrise-web
kubectl get pods
```

---

# Scenario 4 - Readiness Probe Failure

## Problem

The container is running, but Kubernetes considers the application not ready to receive traffic.

The important distinction is:

```text
Running
    !=
Ready
```

A Pod can show:

```text
STATUS:
Running

READY:
0/1
```

---

## Inject the Failure

Change the readiness probe to an invalid path:

```powershell
kubectl patch deployment arrise-web `
  --type=strategic `
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","readinessProbe":{"httpGet":{"path":"/not-ready","port":80}}}]}}}}'
```

Check:

```powershell
kubectl get pods
```

A new Pod may show:

```text
0/1 Running
```

---

## Troubleshoot

Describe the Pod:

```powershell
kubectl describe pod <pod>
```

Look for:

```text
Readiness probe failed
```

Check rollout:

```powershell
kubectl rollout status deployment/arrise-web
```

Check Service backends:

```powershell
kubectl get endpointslice
```

Compare:

```powershell
kubectl get pods -o wide
```

A Pod that fails readiness should not receive normal Service traffic.

---

## Root Cause Flow

```text
Container starts
      |
      v
STATUS = Running
      |
      v
Readiness probe
      |
      X
Probe fails
      |
      v
READY = 0/1
      |
      v
Service does not route traffic to Pod
```

---

## Recover

```powershell
kubectl rollout undo deployment/arrise-web
```

Verify:

```powershell
kubectl rollout status deployment/arrise-web
kubectl get pods
kubectl get endpointslice
```

---

# Scenario 5 - Pods Healthy but Service Is Not Working

## Problem

The Pods are healthy:

```text
1/1 Running
```

The Service also exists.

But the application cannot be reached through the Service.

A common reason is:

```text
Service selector does not match Pod labels
```

---

## Check Healthy Labels First

```powershell
kubectl get pods --show-labels
```

Expected:

```text
app=arrise-web
```

Check the Service:

```powershell
kubectl get svc arrise-web -o yaml
```

Normal selector:

```text
app: arrise-web
```

---

## Inject the Failure

Change the Service selector:

```powershell
kubectl patch service arrise-web `
  -p '{"spec":{"selector":{"app":"wrong-app"}}}'
```

Check Pods:

```powershell
kubectl get pods
```

Pods should remain:

```text
1/1 Running
```

Check Service:

```powershell
kubectl get svc
```

The Service still exists.

Now check:

```powershell
kubectl get endpointslice
```

The Service should no longer have the expected application backends.

---

## Test

```powershell
kubectl run test-client `
  --image=busybox:1.36 `
  --restart=Never `
  --rm -it `
  -- wget -T 5 -qO- http://arrise-web
```

The request should fail.

---

## Troubleshoot

Start with Pods:

```powershell
kubectl get pods
```

Healthy.

Check Service:

```powershell
kubectl get svc arrise-web
```

Exists.

Check endpoints:

```powershell
kubectl get endpointslice
```

No useful backend.

Now compare labels:

```powershell
kubectl get pods --show-labels
```

with Service selector:

```powershell
kubectl get svc arrise-web `
  -o jsonpath="{.spec.selector}"
```

You should find:

```text
Pod label:
app=arrise-web

Service selector:
app=wrong-app
```

---

## Root Cause Flow

```text
Service
   |
selector:
app=wrong-app
   |
   X
Pods:
app=arrise-web
   |
   v
No matching backend
```

This scenario demonstrates:

```text
Pods Running
    !=
Application Reachable
```

---

## Recover

```powershell
kubectl patch service arrise-web `
  -p '{"spec":{"selector":{"app":"arrise-web"}}}'
```

Verify:

```powershell
kubectl get endpointslice
```

Test again:

```powershell
kubectl run test-client `
  --image=busybox:1.36 `
  --restart=Never `
  --rm -it `
  -- wget -qO- http://arrise-web
```

---

# Scenario 6 - Pod Stuck in Pending

## Problem

A Pod remains:

```text
Pending
```

This commonly means the scheduler cannot find a suitable node.

Possible reasons:

```text
Insufficient CPU
Insufficient memory
Node selector mismatch
Affinity rules
Taints / tolerations
PVC/storage issue
```

For this lab we will use a node selector mismatch.

---

## Check Existing Node Labels

```powershell
kubectl get nodes --show-labels
```

We will require a label that does not exist:

```text
lab=does-not-exist
```

---

## Inject the Failure

```powershell
kubectl patch deployment arrise-web `
  --type=strategic `
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"lab":"does-not-exist"}}}}'
```

Check:

```powershell
kubectl get pods
```

New Pods should remain:

```text
Pending
```

---

## Troubleshoot

Describe one Pending Pod:

```powershell
kubectl describe pod <pending-pod>
```

Check:

```text
Events
```

You should see a scheduler-related message.

Check node labels:

```powershell
kubectl get nodes --show-labels
```

Check Pod requirement:

```powershell
kubectl get pod <pending-pod> `
  -o jsonpath="{.spec.nodeSelector}"
```

The mismatch should become clear:

```text
Pod requires:
lab=does-not-exist

Nodes:
No node contains that label
```

---

## Root Cause Flow

```text
Pod
 |
requires:
lab=does-not-exist
 |
Scheduler
 |
checks nodes
 |
No matching node
 |
Pod stays Pending
```

---

## Recover

```powershell
kubectl rollout undo deployment/arrise-web
```

Verify:

```powershell
kubectl rollout status deployment/arrise-web
kubectl get pods
```

---

# Final Health Verification

After finishing all troubleshooting scenarios:

```powershell
kubectl get deployment
```

Expected:

```text
READY 2/2
```

Check Pods:

```powershell
kubectl get pods
```

Expected:

```text
1/1 Running
```

Check Service:

```powershell
kubectl get svc
```

Check endpoints:

```powershell
kubectl get endpointslice
```

Finally test:

```powershell
kubectl run test-client `
  --image=busybox:1.36 `
  --restart=Never `
  --rm -it `
  -- wget -qO- http://arrise-web
```

The application should work normally.

---

# Scenario Summary

| Symptom | Main Area to Check | Important Commands |
|---|---|---|
| `ImagePullBackOff` | Image / registry | `describe pod` |
| `CrashLoopBackOff` | Application / startup | `logs`, `logs --previous`, `describe pod` |
| `CreateContainerConfigError` | Secret / ConfigMap | `describe pod`, `get secrets`, `get configmap` |
| `Running 0/1` | Readiness probe | `describe pod`, `get endpointslice` |
| Pods healthy but Service fails | Selector / endpoints | `get svc`, labels, `get endpointslice` |
| `Pending` | Scheduler / node constraints | `describe pod`, node labels |

---

# Symptom-Based Troubleshooting

```text
ImagePullBackOff
      |
      v
Check image/tag
      |
      v
Check registry authentication
```

```text
CrashLoopBackOff
      |
      v
Check logs
      |
      v
Check previous logs
      |
      v
Check exit code/config/probes
```

```text
CreateContainerConfigError
      |
      v
Check describe events
      |
      v
Check Secret / ConfigMap
```

```text
Running but 0/1 Ready
      |
      v
Check readiness probe
      |
      v
Check EndpointSlice
```

```text
Pods healthy but application unavailable
      |
      v
Check Service
      |
      v
Check selectors
      |
      v
Check EndpointSlice
```

```text
Pending
      |
      v
Check scheduler Events
      |
      v
Check resources
      |
      v
Check labels / selectors / taints
```

---

# Commands to Memorize

```powershell
kubectl get pods -o wide
```

```powershell
kubectl describe pod <pod>
```

```powershell
kubectl logs <pod>
```

```powershell
kubectl logs <pod> --previous
```

```powershell
kubectl get events --sort-by=.lastTimestamp
```

```powershell
kubectl get deployment
```

```powershell
kubectl get svc
```

```powershell
kubectl get pods --show-labels
```

```powershell
kubectl get endpointslice
```

```powershell
kubectl get secrets
```

```powershell
kubectl get configmap
```

```powershell
kubectl get nodes --show-labels
```

```powershell
kubectl top nodes
```

```powershell
kubectl top pods
```

```powershell
kubectl rollout status deployment/arrise-web
```

```powershell
kubectl rollout undo deployment/arrise-web
```

---

# Lab 10 Troubleshooting Mental Model

```text
SYMPTOM
   |
   v
GET
   |
   v
DESCRIBE
   |
   v
EVENTS
   |
   v
LOGS
   |
   v
DEPLOYMENT
   |
   v
SERVICE / ENDPOINTS
   |
   v
CONFIGMAP / SECRET
   |
   v
NODE / RESOURCES
   |
   v
ROOT CAUSE
   |
   v
RECOVER
   |
   v
VERIFY
```

The key objective of this lab is not to memorize individual commands.

The goal is to build a repeatable troubleshooting process that can be applied to different Kubernetes production incidents.
