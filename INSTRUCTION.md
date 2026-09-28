# Kubernetes ToDoApp — DaemonSet and CronJob

## Overview

This project contains Kubernetes manifests for monitoring the TodoApp using:

* **DaemonSet** — periodically sends a request to the TodoApp health endpoint every 5 seconds.
* **CronJob** — sends a request to the TodoApp health endpoint every 4 minutes.

The TodoApp Service is located in the `todoapp` namespace, while the DaemonSet and CronJob are deployed in the `mateapp` namespace.

### Resources

| Resource          | Namespace | Purpose                               |
| ----------------- | --------- | ------------------------------------- |
| `todoapp-service` | `todoapp` | ClusterIP Service of TodoApp          |
| `my-daemonset`    | `mateapp` | Sends health requests every 5 seconds |
| `my-cronjob`      | `mateapp` | Sends health requests every 4 minutes |

---

## Prerequisites

Make sure that:

* Kubernetes cluster is running.
* `kubectl` is installed and configured.
* TodoApp is already deployed.
* `todoapp-service` exists in the `todoapp` namespace.

Check the TodoApp Service:

```bash
kubectl get service todoapp-service -n todoapp
```

Expected result should show a `ClusterIP` Service.

---

## 1. Create the namespace

Create the `mateapp` namespace if it does not already exist:

```bash
kubectl create namespace mateapp
```

If the namespace already exists, Kubernetes will report that it already exists.

Verify it:

```bash
kubectl get namespace mateapp
```

---

## 2. Deploy the DaemonSet

Apply the DaemonSet manifest:

```bash
kubectl apply -f daemonset.yml
```

Check the DaemonSet:

```bash
kubectl get daemonset -n mateapp
```

Check the Pods created by the DaemonSet:

```bash
kubectl get pods -n mateapp -l app=my-daemonset
```

The DaemonSet should create a Pod on each eligible Kubernetes node.

---

## 3. Deploy the CronJob

Apply the CronJob manifest:

```bash
kubectl apply -f cronjob.yml
```

Check the CronJob:

```bash
kubectl get cronjob -n mateapp
```

The CronJob is scheduled to run every 4 minutes.

To see Jobs created by the CronJob:

```bash
kubectl get jobs -n mateapp
```

To see Pods created by these Jobs:

```bash
kubectl get pods -n mateapp
```

---

## 4. Validate the DaemonSet

Check the DaemonSet status:

```bash
kubectl get daemonset my-daemonset -n mateapp
```

A healthy DaemonSet should have the same number of desired and ready Pods.

For more detailed information:

```bash
kubectl describe daemonset my-daemonset -n mateapp
```

Check the DaemonSet Pods:

```bash
kubectl get pods -n mateapp -l app=my-daemonset -o wide
```

---

## 5. Check DaemonSet logs

First, find the Pod created by the DaemonSet:

```bash
kubectl get pods -n mateapp -l app=my-daemonset
```

Then view its logs:

```bash
kubectl logs <daemonset-pod-name> -n mateapp
```

Replace `<daemonset-pod-name>` with the actual Pod name.

The DaemonSet continuously executes the health request:

```text
curl → TodoApp /api/health
        ↓
     sleep 5
        ↓
curl → TodoApp /api/health
        ↓
     sleep 5
        ↓
       ...
```

---

## 6. Validate the CronJob

Check the CronJob configuration:

```bash
kubectl get cronjob my-cronjob -n mateapp
```

For detailed information:

```bash
kubectl describe cronjob my-cronjob -n mateapp
```

Check Jobs created by the CronJob:

```bash
kubectl get jobs -n mateapp
```

The CronJob should create a new Job every 4 minutes.

---

## 7. Check CronJob logs

A CronJob creates Jobs, and Jobs create Pods. Therefore, logs must be checked on the Pod created by the Job.

List the Pods:

```bash
kubectl get pods -n mateapp
```

Find the Pod created by the CronJob and check its logs:

```bash
kubectl logs <cronjob-pod-name> -n mateapp
```

Replace `<cronjob-pod-name>` with the actual Pod name.

The CronJob performs a single request to:

```text
/api/health
```

and then the Pod finishes.

---

## 8. Check all resources

To see the resources deployed in the `mateapp` namespace:

```bash
kubectl get all -n mateapp
```

You can also check the CronJob separately:

```bash
kubectl get cronjob -n mateapp
```

and the DaemonSet separately:

```bash
kubectl get daemonset -n mateapp
```

---

## 9. Cleanup

To remove the DaemonSet:

```bash
kubectl delete -f daemonset.yml
```

To remove the CronJob:

```bash
kubectl delete -f cronjob.yml
```

The `mateapp` namespace can be removed after all resources are no longer needed:

```bash
kubectl delete namespace mateapp
```

---

## Expected Behavior

### DaemonSet

The DaemonSet runs continuously on each eligible node.

Its container:

1. Sends a request to the TodoApp ClusterIP Service.
2. Calls the `/api/health` endpoint.
3. Waits 5 seconds.
4. Repeats the request.

### CronJob

The CronJob:

1. Starts every 4 minutes.
2. Creates a Job.
3. The Job creates a Pod.
4. The Pod sends a request to the TodoApp `/api/health` endpoint.
5. The request finishes and the Pod exits.
6. Kubernetes keeps the configured Job history.

The CronJob uses:

* `concurrencyPolicy: Allow`
* `successfulJobsHistoryLimit: 10`
* `failedJobsHistoryLimit: 5`
* `busyboxplus:curl` image
* configured CPU and memory requests/limits.
