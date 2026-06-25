# Kubernetes Deployments - LO4

## Create a Deployment

**Q:** Create a Deployment named `web` running `nginx:1.25` with 3 replicas using a single imperative `kubectl create deployment` command, then export its generated YAML to a file.

**A:**

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web-deployment.yaml
```

---

## DockerHub Authentication

**Q:** Enable pulling images from DockerHub as an authenticated user to lower the chance of hitting the pull limit. What kind of resource do you need to create?

**A:**

Create a Kubernetes Secret of type `docker-registry` (`kubernetes.io/dockerconfigjson`) image pull secret.

---

## Scale a Deployment

**Q:** Scale `web` from 3 to 5 replicas two different ways — once with `kubectl scale`, once by editing the manifest.

**A:**

```bash
kubectl scale deployment web --replicas=5
```

or edit the manifest:

```yaml
spec:
  replicas: 5
```

Apply:

```bash
kubectl apply -f web-deployment.yaml
```

---

## Rolling Update

**Q:** Perform a rolling update of `web` from `nginx:1.25` to `nginx:1.27` and watch it with `kubectl rollout status`.

**A:**

```bash
kubectl set image deployment/web nginx=docker.io/library/nginx:1.27
kubectl rollout status deployment/web
```

---

## Rollout History and Rollback

**Q:** View a Deployment's rollout history and roll back to the previous revision. Explain what the `CHANGE-CAUSE` column shows and how to populate it.

**A:**

View history:

```bash
kubectl rollout history deployment/web
```

Add change cause:

```bash
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade nginx 1.25 -> 1.27" \
  --overwrite
```

Rollback:

```bash
kubectl rollout undo deployment/web
```

`CHANGE-CAUSE` shows the reason for the revision. If it shows `<none>`, no annotation was added.

---

## RollingUpdate Strategy

**Q:** Set the update strategy to `RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 0`, and explain the effect on availability during an update.

**A:**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

`maxSurge: 1` allows one extra Pod during updates, while `maxUnavailable: 0` ensures no Pods are unavailable, enabling zero-downtime updates.

---

## Recreate Strategy

**Q:** Switch a Deployment's strategy to `Recreate` and describe a concrete scenario where this is required instead of `RollingUpdate`.

**A:**

```yaml
spec:
  strategy:
    type: Recreate
```

All old Pods are terminated before new Pods are created. Useful when old and new application versions cannot run simultaneously (e.g., incompatible database schema changes).

---

## Resource Requests and Limits

**Q:** Add CPU/memory requests and limits to a Deployment's container and verify they appear in the running Pod spec.

**A:**

```bash
kubectl set resources deployment/web \
  -c nginx \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi
```

Verify:

```bash
kubectl describe pod <pod-name>
```

---

## Revision History Limit

**Q:** Set `revisionHistoryLimit: 3` and explain how it affects your ability to roll back and how many old ReplicaSets are kept.

**A:**

```yaml
spec:
  revisionHistoryLimit: 3
```

Only the three most recent previous revisions are retained. Rollback is only possible to revisions that still exist.

---

## Failed Image Rollout

**Q:** Set a Deployment's image to a non-existent tag, observe the rollout get stuck, and explain why the old Pods keep serving traffic.

**A:**

```bash
kubectl set image deployment/web nginx=nginx:does-not-exist
```

The new Pods cannot start because the image tag does not exist. The old Pods continue serving traffic until replacement Pods become Ready.

---

## Label Selectors

**Q:** Use a label selector to list only the Pods belonging to one Deployment, and explain the Deployment → ReplicaSet → Pod selector relationship.

**A:**

```bash
kubectl get pods -l app=web
```

Deployment → ReplicaSet → Pods. The Deployment manages a ReplicaSet using label selectors, and the ReplicaSet manages Pods with matching labels.

---

## Expose a Deployment

**Q:** Expose a Deployment with `kubectl expose`, then explain what object was created and how its selector was derived.

**A:**

```bash
kubectl expose deployment web --port=80 --target-port=80
```

Creates a Service (default type: ClusterIP). The selector is automatically derived from the Deployment's Pod labels.

---

## Sidecar Container

**Q:** Add a sidecar (second) container to a Deployment's Pod template and explain how the two containers share the Pod network and volumes.

**A:**

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.27

    - name: sidecar
      image: busybox
      command:
        - sh
        - -c
        - while true; do sleep 30; done
```

Containers in the same Pod share the same IP address, network namespace, and any mounted volumes.

---

## Deployment with nodeSelector

**Q:** Write a Deployment manifest for `httpd:2.4` with 2 replicas, a named container port, and a `nodeSelector`; explain why it stays Pending if no node matches.

**A:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: httpd

  template:
    metadata:
      labels:
        app: httpd

    spec:
      nodeSelector:
        disktype: ssd

      containers:
        - name: httpd
          image: httpd:2.4

          ports:
            - name: http
              containerPort: 80
```

If no node has the label `disktype=ssd`, the scheduler cannot place the Pods and they remain in the `Pending` state.

---

## Bare Pod vs Deployment-Managed Pod

**Q:** Create a bare Pod (no controller) running BusyBox that sleeps, then explain what happens when you delete it versus a Deployment-managed Pod.

**A:**

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: busybox-sleep

spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3600"
```

Create:

```bash
kubectl apply -f busybox-pod.yaml
```

A bare Pod is permanently removed when deleted. A Deployment-managed Pod is automatically recreated by its ReplicaSet.
