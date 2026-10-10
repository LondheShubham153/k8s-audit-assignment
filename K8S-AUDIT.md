# GreenCart Kubernetes Audit

**Audit Date:** October 10, 2026
**Namespace:** `greencart`
**Repository:** [sameersyed7/GreenCart](https://github.com/sameersyed7/GreenCart/tree/main)
**Kubernetes Manifests:** [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s)

## Audit Summary

```text
GreenCart Kubernetes Audit
Namespace: greencart
Repository: /home/ubuntu/greenapp/GreenCart

PASS: 15
WARN: 1
FAIL: 0
```

**Overall result:** 15 PASS, 1 WARN, 0 FAIL.

The only warning is that a GitHub Actions workflow for deploying to a `kind` cluster was not detected. This is a stretch goal.

---

## My App

* **Name:** GreenCart
* **Repository:** https://github.com/sameersyed7/GreenCart/tree/main
* **Architecture:** Three-tier application.

  * Frontend: React and Vite
  * Backend: Express.js
  * Database: MongoDB
* **Kubernetes manifests:** `k8s/`
* **Fresh-cluster instructions:** `runfromscratch.txt`
* **Cluster context used for audit:** `kind-audit`
* **Cluster:** One control-plane node and two worker nodes.

To set up the application on a fresh machine, follow the commands in `runfromscratch.txt`, starting with the `kind create cluster` step. Follow the instructions in that file to access the application using its URL or port-forward command.


# Evidence

## 1. Cluster and Workload Status

**Commands:**

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -n greencart -o wide
kubectl get deployments,replicasets,services -n greencart
```

**Observed results:**

* Current context: `kind-audit`
* Three cluster nodes are Ready:

  * `audit-control-plane`
  * `audit-worker`
  * `audit-worker2`
* Backend Deployment: 1/1 replicas available.
* Frontend Deployment: 2/2 replicas available.
* MongoDB Deployment: 1/1 replicas available.
* Frontend, backend, and MongoDB Pods are Running and Ready.
* The audit CronJob Pod shown in the output is Completed.

**Status: PASS**

**Reason:** All three cluster nodes are Ready, and the frontend, backend, and MongoDB Deployments have their desired replicas available.

## 2. Service Endpoints

**Command:**

```bash
kubectl get services,endpointslices -n greencart
```

**Observed results:**

| Service            |  Port | Endpoint addresses         |
| ------------------ | ----: | -------------------------- |
| `backend`          |  4000 | `10.244.1.6`               |
| `frontend-service` |    80 | `10.244.2.9`, `10.244.1.8` |
| `mongodb`          | 27017 | `10.244.2.6`               |

**Status: PASS**

**Reason:** Each application Service has EndpointSlice addresses pointing to its target Pods, enabling Kubernetes service discovery.

## 3. Rolling Update and Rollback

**Commands:**

```bash
kubectl rollout history deployment/frontend-deployment -n greencart
kubectl rollout status deployment/frontend-deployment -n greencart
```

**Observed results:**

* Rollout history contains revisions 1, 3, and 4.
* The rollout status command reported:

```text
deployment "frontend-deployment" successfully rolled out
```

**Status: PASS for rollout; rollback evidence incomplete.**

**Reason:** The frontend Deployment has rollout history and its current rollout completed successfully.

**Important:** This evidence does not prove that a rollback was performed. A Deployment revision can be created by changing the Pod template without changing the application image. If rollback is required, capture the rollout history and status before and after an actual rollback.

Useful commands:

```bash
kubectl rollout history deployment/frontend-deployment -n greencart

kubectl rollout undo deployment/frontend-deployment -n greencart

kubectl rollout status deployment/frontend-deployment -n greencart

kubectl rollout history deployment/frontend-deployment -n greencart
```

Only report a successful rollback after verifying the actual results.

## 4. Secret Existence

**Command:**

```bash
kubectl get secret greencart-secrets -n greencart
```

**Observed result:**

```text
NAME                TYPE     DATA   AGE
greencart-secrets   Opaque   8      94m
```

**Status: PASS**

**Reason:** The `greencart-secrets` Kubernetes Secret exists and contains eight data entries.

**Security note:** Never include unredacted Secret values in screenshots, documentation, or GitHub commits. Avoid displaying Secret YAML containing encoded data.

## 5. RBAC and ServiceAccount Permissions

**Commands:**

```bash
kubectl auth can-i list pods -n greencart \
  --as=system:serviceaccount:greencart:greencart-auditor

kubectl auth can-i delete pods -n greencart \
  --as=system:serviceaccount:greencart:greencart-auditor
```

**Observed results:**

```text
yes
no
```

**Status: PASS**

**Reason:** The auditor ServiceAccount can list Pods but cannot delete them, demonstrating least-privilege access for these permission checks.

## 6. Ingress Test

**Command:**

```bash
curl -sS -o /dev/null -w 'HTTP status: %{http_code}\n' \
  -H 'Host: greencart.local' http://127.0.0.1/
```

**Observed result:**

```text
HTTP status: 200
```

**Status: PASS**

**Reason:** The request using the `greencart.local` Host header returned HTTP 200, confirming that the tested Ingress route reaches the frontend successfully.

## 7. CronJob Evidence

**Commands:**

```bash
kubectl get cronjobs,jobs -n greencart
kubectl logs job/greencart-audit-test -n greencart
```

**Observed results:**

* CronJob: `greencart-audit-cronjob`
* Schedule: `*/5 * * * *`
* A generated Job completed successfully.
* The observed Job completed 1/1 with a duration of five seconds.

**Status: PASS for CronJob existence and Job completion.**

**Reason:** The CronJob is configured to run every five minutes, and the captured Job status confirms that a scheduled task completed successfully.

**Note:** The captured Job listing shows a generated Job named `greencart-audit-cronjob-29860150`, while the example log command references `greencart-audit-test`. Check the actual Job name before collecting logs.

Use:

```bash
kubectl get jobs -n greencart
kubectl logs job/greencart-audit-cronjob-29860150 -n greencart
```

Replace the example Job name with the current completed Job name if it has changed.

---

# Audit Checklist

| Concept                                    | Status              | Evidence                                                                                                                             | Why I used it in GreenCart                                                                                           | Where to look                                                                                                                                                                                                                                                                                         |
| ------------------------------------------ | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Deployment and ReplicaSet                  | ✅ PASS              | Deployment output showed frontend 2/2, backend 1/1, MongoDB 1/1, and multiple ReplicaSets.                                           | Deployments maintain the desired number of Pods, while ReplicaSets manage Pod replicas.                              | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Deployment example](https://github.com/LondheShubham153/kubestarter/blob/main/examples/nginx/deployment.yml) · [ReplicaSet example](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/replicasets.yml) |
| Service                                    | ✅ PASS              | Services and EndpointSlices showed endpoint addresses for frontend, backend, and MongoDB.                                            | Services provide stable networking to Pods, even when Pod IP addresses change.                                       | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Service example](https://github.com/LondheShubham153/kubestarter/blob/main/examples/nginx/service.yml)                                                                                                                            |
| Namespace                                  | ✅ PASS              | Audit script confirmed that namespace `greencart` exists.                                                                            | A namespace groups GreenCart resources and separates them from resources in other namespaces.                        | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Namespace documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)                                                                                                                           |
| Labels and selectors                       | ⚠️ VERIFY           | EndpointSlices show Service endpoints, but the actual label and selector definitions were not included in the captured evidence.     | Labels and selectors allow Services and controllers to identify the correct GreenCart Pods.                          | Inspect `metadata.labels` and `spec.selector` in [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s). [Labels documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)                                                                                  |
| Rolling update and rollback                | ⚠️ PARTIAL EVIDENCE | Rollout history listed revisions 1, 3, and 4; rollout status succeeded. No before-and-after rollback evidence was captured.          | Rolling updates deploy changes gradually, while rollback helps restore a previous Deployment revision.               | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Deployment documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)                                                                                                                                  |
| ConfigMap                                  | ✅ PASS              | Audit script confirmed ConfigMap resources exist.                                                                                    | ConfigMaps separate non-sensitive application configuration from container images.                                   | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [ConfigMap example](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/configMap.yml)                                                                                                                        |
| Secret                                     | ✅ PASS              | `greencart-secrets` exists as an Opaque Secret with eight data entries.                                                              | Secrets provide a Kubernetes resource for sensitive configuration.                                                   | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Secret example](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/secrets.yml)                                                                                                                             |
| Requests and limits                        | ✅ PASS              | Audit script confirmed CPU and memory requests and limits on all Deployments.                                                        | Requests help Kubernetes schedule Pods, while limits constrain resource consumption.                                 | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)                                                                                                                              |
| Liveness and readiness probes              | ✅ PASS              | Audit script confirmed that all Deployments have both probes.                                                                        | Readiness probes help prevent traffic from reaching unready Pods; liveness probes help detect unhealthy containers.  | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Probe documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)                                                                                                       |
| PersistentVolumeClaim                      | ✅ PASS              | Audit script confirmed that a PVC exists.                                                                                            | Persistent storage can preserve database data when Pods are replaced, depending on the configured storage lifecycle. | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [PVC example](https://github.com/LondheShubham153/kubestarter/blob/main/PersistentVolumes/PersistentVolumeClaim.yaml)                                                                                                              |
| Ingress                                    | ✅ PASS              | Audit detected an Ingress resource and running controller; the HTTP test returned 200.                                               | Ingress routes incoming HTTP traffic to the frontend using configured routing rules.                                 | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [Ingress examples](https://github.com/LondheShubham153/kubestarter/blob/main/Ingress/)                                                                                                                                             |
| Multi-node kind cluster                    | ✅ PASS              | `kubectl get nodes -o wide` showed three Ready nodes.                                                                                | A multi-node cluster allows me to test Kubernetes scheduling across a control plane and worker nodes.                | [`kind/kind-config.yaml`](kind/kind-config.yaml) · [kind cluster example](https://github.com/LondheShubham153/kubestarter/blob/main/kind-cluster/kind-config.yml)                                                                                                                                     |
| HPA (stretch)                              | ✅ PASS              | Audit script confirmed that a HorizontalPodAutoscaler exists.                                                                        | HPA can automatically adjust replica counts based on configured metrics and targets.                                 | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [HPA example](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/apache-hpa.yml)                                                                                                                                    |
| RBAC and ServiceAccount (stretch)          | ✅ PASS              | Audit confirmed ServiceAccount, Role, and RoleBinding. Permission checks returned `yes` for listing Pods and `no` for deleting Pods. | RBAC limits the actions a service account can perform and follows the principle of least privilege.                  | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [RBAC examples](https://github.com/LondheShubham153/kubestarter/tree/main/RBAC)                                                                                                                                                    |
| CronJob (stretch)                          | ✅ PASS              | Audit confirmed a CronJob exists; the captured generated Job completed 1/1.                                                          | CronJobs automate repeatable tasks on a schedule.                                                                    | [`k8s/`](https://github.com/sameersyed7/GreenCart/tree/main/k8s) · [CronJob documentation](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)                                                                                                                                      |
| GitHub Actions deploying to kind (stretch) | ⚠️ WARN             | Audit script reported that a GitHub Actions `kind` workflow was not detected.                                                        | CI automation can validate manifests and test Kubernetes deployment changes consistently.                            | [helm/kind-action](https://github.com/helm/kind-action)                                                                                                                                                                                                                                               |

---

# Score

**Audit script result:** 15 PASS, 1 WARN, 0 FAIL.

* **Must-have concepts with PASS and evidence (maximum 12):** Review the core concepts above. Capture actual labels/selectors and rollback evidence if required by the assignment.
* **Stretch concepts with PASS and evidence (maximum 4):** HPA, RBAC/ServiceAccount, and CronJob are reported as passing. Keep their manifests and supporting command output available.
* **Total score:** Calculate according to the assignment's scoring rules after checking the evidence for each required concept.

The audit script's PASS count is not necessarily the same as the assignment score. Count only the concepts for which the required evidence is available.

---

# What Was Hard / What I Would Change



One challenge was getting the frontend, backend, and MongoDB components to communicate correctly inside Kubernetes. I learned to troubleshoot Pods, Services, EndpointSlices, and Ingress routing rather than assuming that a running Pod means the entire application is working.

I also learned how Kubernetes Deployments manage replicas, how ConfigMaps and Secrets separate configuration from application code, and how RBAC restricts access.

If I were to improve this project, I would add a GitHub Actions workflow to validate the manifests and test the deployment automatically. I would also document a clear rollback demonstration and keep the setup instructions updated so the application can be deployed on a fresh cluster.

---

# Final Verification Commands

Run these commands from the GreenCart repository root before submission.

```bash
./audit.sh greencart

kubectl config current-context
kind-audit

kubectl get nodes -o wide
NAME                  STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION           CONTAINER-RUNTIME
audit-control-plane   Ready    control-plane   4h30m   v1.36.1   172.20.0.3    <none>        Debian GNU/Linux 13 (trixie)   7.0.0-1014-aws (amd64)   containerd://2.3.1
audit-worker          Ready    <none>          4h29m   v1.36.1   172.20.0.4    <none>        Debian GNU/Linux 13 (trixie)   7.0.0-1014-aws (amd64)   containerd://2.3.1
audit-worker2         Ready    <none>          4h29m   v1.36.1   172.20.0.2    <none>        Debian GNU/Linux 13 (trixie)   7.0.0-1014-aws (amd64)   containerd://2.3.1



kubectl get pods -n greencart -o wide
NAME                                     READY   STATUS      RESTARTS   AGE    IP            NODE            NOMINATED NODE   READINESS GATES
backend-deployment-86cbdd7c75-hm28p      1/1     Running     0          122m   10.244.1.6    audit-worker2   <none>           <none>
frontend-deployment-86cd8794d4-68rlf     1/1     Running     0          109m   10.244.2.9    audit-worker    <none>           <none>
frontend-deployment-86cd8794d4-9kslv     1/1     Running     0          109m   10.244.1.8    audit-worker2   <none>           <none>
greencart-audit-cronjob-29860180-75d8d   0/1     Completed   0          61s    10.244.2.31   audit-worker    <none>           <none>
mongodb-bd77c88c8-n9lx5                  1/1     Running     0          139m   10.244.2.6    audit-worker    <none>           <none>




kubectl get deployments,replicasets,services -n greencart
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-deployment    1/1     1            1           123m
deployment.apps/frontend-deployment   2/2     2            2           4h1m
deployment.apps/mongodb               1/1     1            1           140m

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-deployment-86cbdd7c75    1         1         1       123m
replicaset.apps/frontend-deployment-747cd676cd   0         0         0       4h1m
replicaset.apps/frontend-deployment-74f969554b   0         0         0       111m
replicaset.apps/frontend-deployment-86cd8794d4   2         2         2       135m
replicaset.apps/mongodb-bd77c88c8                1         1         1       140m

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/backend            ClusterIP   10.96.207.120   <none>        4000/TCP       126m
service/frontend-service   NodePort    10.96.46.208    <none>        80:30001/TCP   3h57m
service/mongodb            ClusterIP   10.96.34.83     <none>        27017/TCP      140m



kubectl get services,endpointslices -n greencart
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/backend            ClusterIP   10.96.207.120   <none>        4000/TCP       126m
service/frontend-service   NodePort    10.96.46.208    <none>        80:30001/TCP   3h58m
service/mongodb            ClusterIP   10.96.34.83     <none>        27017/TCP      140m

NAME                                                    ADDRESSTYPE   PORTS   ENDPOINTS               AGE
endpointslice.discovery.k8s.io/backend-hlp7m            IPv4          4000    10.244.1.6              126m
endpointslice.discovery.k8s.io/frontend-service-gznxz   IPv4          80      10.244.2.9,10.244.1.8   3h58m
endpointslice.discovery.k8s.io/mongodb-22b8p            IPv4          27017   10.244.2.6              140m




kubectl get configmaps,secrets,pvc,ingress,hpa,cronjobs -n greencart
NAME                            DATA   AGE
configmap/greencart-configmap   2      150m
configmap/kube-root-ca.crt      1      4h10m
configmap/mongodb-configmap     1      150m

NAME                       TYPE     DATA   AGE
secret/greencart-secrets   Opaque   8      125m

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mongodb-pvc   Bound    pvc-cdaed3f3-b536-42a0-938e-f09a73b45f1b   1Gi        RWO            standard       <unset>                 141m

NAME                                          CLASS   HOSTS             ADDRESS     PORTS   AGE
ingress.networking.k8s.io/greencart-ingress   nginx   greencart.local   localhost   80      108m

NAME                                               REFERENCE                        TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/frontend-hpa   Deployment/frontend-deployment   cpu: 10%/70%   2         3         2          98m

NAME                                    SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/greencart-audit-cronjob   */5 * * * *   <none>     False     0        2m24s           104m




kubectl rollout history deployment/frontend-deployment -n greencart
deployment.apps/frontend-deployment
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>



kubectl rollout status deployment/frontend-deployment -n greencart
deployment "frontend-deployment" successfully rolled out



kubectl get jobs -n greencart
NAME                               STATUS     COMPLETIONS   DURATION   AGE
greencart-audit-cronjob-29860180   Complete   1/1           5s         3m34s



curl -sS -o /dev/null -w 'HTTP status: %{http_code}\n' \
  -H 'Host: greencart.local' http://127.0.0.1/
HTTP status: 200

```

Verify that the final command output matches the claims in this document. Redact sensitive information before sharing screenshots or publishing evidence.
