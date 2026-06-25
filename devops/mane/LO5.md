# Containers & Kubernetes Troubleshooting Cheat Sheet

## Exam Questions and Answers

### LO5 --- Solve problems with application shipping by using containers

> **Task / What you need to explain:** nginx listens on 80; the site
> isn't reachable on the host port you expected --- what's reversed?

``` bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
```

``` bash
podman ps
```

``` bash
podman port web
```

``` bash
podman logs web
```

``` bash
podman exec web ss -tln
```

------------------------------------------------------------------------

## Question 2

**Full Question:** podman run -d --name db -e MYSQL_ROOT_PASSWORD
docker.io/library/mysql

> **Task / What you need to explain:** the container exits during init
> --- inspect podman logs.

``` bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD=MySecret123 docker.io/library/mysql
```

``` bash
podman ps -a
```

``` bash
podman logs db
```

------------------------------------------------------------------------

## Question 3

**Full Question:** podman run -d --name app --network host -p 8080:80
docker.io/library/nginx

> **Task / What you need to explain:** why is the -p flag effectively
> ignored here?

``` bash
podman run -d --name app --network host docker.io/library/nginx
```

``` bash
podman run -d --name app -p 8080:80 docker.io/library/nginx
```

Because --network host bypasses network isolation and port forwarding

------------------------------------------------------------------------

## Question 4

**Full Question:** podman run -d --name c1 docker.io/library/busybox

> **Task / What you need to explain:** the container goes straight to
> Exited (0) --- why, and how do you keep it running?

``` bash
podman run -d --name c1 docker.io/library/busybox sleep 1d
```

``` bash
podman run -d --name c1 docker.io/library/busybox tail -f /dev/null
```

``` bash
podman ps -a
```

``` bash
podman logs c1
```

------------------------------------------------------------------------

## Question 5

**Full Question:** podman run --rm -d --name job
docker.io/library/alpine echo hello

> **Task / What you need to explain:** then podman logs job fails ---
> explain the --rm + detached gotcha.

The command echo hello finishes immediately. Since --rm is used, Podman
automatically removes the container after it exits. Therefore, podman
logs job fails because the container no longer exists.

``` bash
podman run -d --name job docker.io/library/alpine echo hello
```

------------------------------------------------------------------------

## Question 6

**Full Question:** podman run -d -p 8080:80 --memory 8m
docker.io/library/mysql

> **Task / What you need to explain:** the database never becomes
> healthy --- what limit is the problem?

The memory limit (--memory 8m) is too low. MySQL requires much more than
8 MB to start, so the container cannot initialize and never becomes
healthy.

``` bash
podman run -d --memory 512m docker.io/library/mysql or podman run -d docker.io/library/mysql
```

------------------------------------------------------------------------

## Question 7

**Full Question:** podman run -d --name a docker.io/library/alpine sleep
1d

``` bash
podman run -d --name b docker.io/library/alpine sleep 1d
```

``` bash
podman exec a ping b
```

> **Task / What you need to explain:** name resolution fails --- why, on
> the default network, and how do you fix it?

Containers on the default network cannot resolve each other's names. DNS
resolution for container names works only on a user-defined network.

``` bash
podman network create mynet
```

``` bash
podman run -d --name a --network mynet docker.io/library/alpine sleep 1d
```

``` bash
podman run -d --name b --network mynet docker.io/library/alpine sleep 1d
```

``` bash
podman exec a ping b
```

------------------------------------------------------------------------

## Question 8

**Full Question:** podman run -d --name web -v
./html:/usr/share/nginx/html docker.io/library/nginx

> **Task / What you need to explain:** the files aren't visible in the
> container, or SELinux denies access --- what mount flag is missing?

``` bash
podman run -d --name web \
```

-v ./html:/usr/share/nginx/html:Z\

docker.io/library/nginx

### Explanation

:Z → Private SELinux label (used by one container).

:z → Shared SELinux label (used by multiple containers).

Without these flags, the container may not be able to read or write the
mounted files.

------------------------------------------------------------------------

## Question 9

**Full Question:** For each Containerfile/Dockerfile: identify the
mistake, fix it, and explain what was wrong.

------------------------------------------------------------------------

## Question 10

**Full Question:** FROM debian:12

``` bash
RUN apt-get install -y nginx
```

> **Task / What you need to explain:** the build fails to find the
> package --- what's missing before install?

apt-get install is executed without first updating the package index. As
a result, the build may fail because the package lists are outdated or
missing.

``` bash
FROM debian:12
```

``` bash
RUN apt-get update && apt-get install -y nginx
```

------------------------------------------------------------------------

## Question 11

**Full Question:** FROM python:3.11

``` bash
COPY app.py /app/app.py
```

WORKDIR /app

``` bash
CMD python app.py
```

> **Task / What you need to explain:** the app starts but doesn't handle
> signals / can't be stopped cleanly --- shell vs exec form?

``` bash
FROM python:3.11
```

``` bash
COPY app.py /app/app.py
```

WORKDIR /app

``` bash
CMD ["python", "app.py"]
```

``` bash
CMD python app.py uses the shell form. The shell becomes PID 1, so Python does not receive signals (e.g., SIGTERM) properly, making the container harder to stop cleanly.
```

------------------------------------------------------------------------

## Question 12

**Full Question:** FROM node:20

``` bash
COPY . /app
```

WORKDIR /app

``` bash
RUN npm install
```

> **Task / What you need to explain:** every source change triggers a
> full npm install --- reorder for layer caching.

``` bash
COPY . /app copies all source files before npm install. Any change in the source code invalidates the Docker cache, causing npm install to run again.
```

``` bash
FROM node:20
```

WORKDIR /app

``` bash
COPY package*.json ./
```

``` bash
RUN npm install
```

``` bash
COPY . .
```

package.json changes less often than the application code.

Docker reuses the cached npm install layer if dependencies haven't
changed.

Only the final COPY . . layer is rebuilt when source files change

------------------------------------------------------------------------

## Question 13

**Full Question:** FROM alpine:3.20

``` bash
ENV PATH=/app/bin
```

``` bash
RUN apk add --no-cache curl
```

> **Task / What you need to explain:** after this, curl/sh aren't found
> --- what did setting PATH break?

PATH is overwritten with /app/bin. System directories like /usr/bin and
/bin are removed, so commands such as curl and sh cannot be found.

``` bash
FROM alpine:3.20
```

``` bash
ENV PATH="/app/bin:${PATH}"
```

``` bash
RUN apk add --no-cache curl
```

------------------------------------------------------------------------

## Question 14

**Full Question:** FROM ubuntu:24.04

``` bash
EXPOSE 8080
```

``` bash
CMD ["python3", "-m", "http.server", "3000"]
```

The application listens on port 3000, but EXPOSE declares port 8080.
This mismatch can cause confusion when mapping ports (e.g., -p 8080:8080
won't work).

``` bash
FROM ubuntu:24.04
```

``` bash
EXPOSE 3000
```

``` bash
CMD ["python3", "-m", "http.server", "3000"]
```

------------------------------------------------------------------------

## Question 15

**Full Question:** (you map -p 8080:8080 but nothing answers --- what
mismatch is there, and what does EXPOSE actually

do?)

Documents which port the application is expected to use.

Helps with readability and some container tools.

Does not publish or open the port

------------------------------------------------------------------------

## Question 16

**Full Question:** FROM debian:12

``` bash
RUN apt-get update && apt-get install -y build-essential
```

> **Task / What you need to explain:** the image is huge --- how do you
> cut the size in the same layer?

``` bash
FROM debian:12
```

``` bash
RUN apt-get update && \
```

apt-get install -y build-essential &&\

apt-get clean &&\

rm -rf /var/lib/apt/lists/\*

The image is large because the APT package cache is left inside the
image after installation.

------------------------------------------------------------------------

## Question 17

**Full Question:** FROM python:3.11

``` bash
USER appuser
```

``` bash
COPY requirements.txt /app/requirements.txt
```

``` bash
RUN pip install -r /app/requirements.txt
```

> **Task / What you need to explain:** permission denied errors ---
> what's wrong with the USER/COPY ordering and ownership?

``` bash
USER appuser is set before copying files and installing packages. The non-root user may not have permission to write to /app or install Python packages, causing Permission denied errors.
```

``` bash
FROM python:3.11
```

``` bash
COPY requirements.txt /app/requirements.txt
```

``` bash
RUN pip install -r /app/requirements.txt
```

``` bash
USER appuser
```

------------------------------------------------------------------------

## Question 18

**Full Question:** FROM golang:1.22

``` bash
COPY . /src
```

WORKDIR /src

``` bash
RUN go build -o app .
```

``` bash
CMD ["/src/app"]
```

> **Task / What you need to explain:** the final image is \~1 GB --- how
> would a multi-stage build fix it?

The final image contains the Go compiler, build tools, source code, and
dependencies, making it very large (\~1 GB).

# Build stage

``` bash
FROM golang:1.22 AS builder
```

WORKDIR /src

``` bash
COPY . .
```

``` bash
RUN go build -o app .
```

# Runtime stage

``` bash
FROM alpine:3.20
```

``` bash
COPY --from=builder /src/app /app
```

``` bash
CMD ["/app"]
```

------------------------------------------------------------------------

## Question 19

**Full Question:** A pod is stuck Pending. Walk through the diagnosis
with kubectl describe pod and identify the reason from

the events (scheduling / resources / PVC / nodeSelector).

Which command is used to diagnose a Pending Pod?

A:

``` bash
kubectl describe pod <pod-name>
```

**Q: Where do you find the cause?**

A: In the Events section of kubectl describe pod.

**Q: Name three common reasons for a Pod being Pending.**

A:

Insufficient CPU or memory.

Missing or unbound PVC.

nodeSelector does not match any node.

------------------------------------------------------------------------

## Question 20

**Full Question:** Find a deployment from older tasks, remove a couple
of spaces and try to deploy it. Identify the cause in the

error messages and fix it.

You accidentally remove a few spaces from a Deployment YAML file. Since
YAML is indentation-sensitive, Kubernetes cannot parse the manifest.

YAML uses spaces to define structure. Missing or incorrect indentation
changes the hierarchy, making the file invalid. YAML needs good spacing

------------------------------------------------------------------------

## Question 21

**Full Question:** A pod is in ImagePullBackOff. Diagnose the cause
(image typo, missing tag, private registry without pull

secret) and fix it.

A Pod is in ImagePullBackOff because Kubernetes cannot pull the
container image.

Image name typo.

Invalid or missing image tag.

Private registry without an imagePullSecret.

------------------------------------------------------------------------

## Question 22

**Full Question:** A pod is in CrashLoopBackOff. Use kubectl logs
--previous to read the last crash and fix the root cause.

The Pod is in CrashLoopBackOff because the container starts, crashes,
and Kubernetes keeps restarting it.

``` bash
kubectl logs --previous <pod-name>
```

CrashLoopBackOff means the application inside the container keeps
crashing. Use kubectl logs --previous to identify the error, fix the
underlying issue (configuration, command, environment variables, missing
files, etc.), and redeploy.

------------------------------------------------------------------------

## Question 23

**Full Question:** A pod's last state shows OOMKilled. Confirm it from
kubectl get pod -o yaml / describe, then fix it (raise the

memory limit or fix the app).

``` bash
kubectl describe pod <pod>
```

``` bash
kubectl get pod <pod> -o yaml
```

Last State: Terminated

Reason: OOMKilled

### Fix

Increase memory limit.

Optimize the application.

------------------------------------------------------------------------

## Question 24

**Full Question:** A Deployment's pods never become Ready. Trace it to a
failing readiness probe and correct the probe or

the app.

``` bash
kubectl describe pod <pod>
```

``` bash
kubectl logs <pod>
```

Correct the readiness probe.

Ensure the application listens on the correct port/path.

------------------------------------------------------------------------

## Question 25

**Full Question:** A Service returns nothing. Diagnose a Service/pod
label mismatch using kubectl get endpoints.

``` bash
kubectl get svc
```

``` bash
kubectl get pods --show-labels
```

``` bash
kubectl get endpoints
```

If:

ENDPOINTS: `<none>`{=html}

→ labels don't match.

### Fix

Match Service selector with Pod labels.

------------------------------------------------------------------------

## Question 26

**Full Question:** Given a manifest where spec.selector.matchLabels
doesn't match spec.template.metadata.labels, explain

the error kubectl apply returns and fix it.

### Problem

``` bash
selector:
```

matchLabels:

app: web

``` bash
template:
```

metadata:

labels:

app: nginx

Error

selector does not match template labels

### Fix

Both labels must be identical.

------------------------------------------------------------------------

## Question 27

**Full Question:** Given a manifest whose resources.requests exceed any
node's capacity, explain why the pod is Pending

(Insufficient cpu/memory) and fix it.

### Diagnose

``` bash
kubectl describe pod <pod>
```

Example:

Insufficient memory

### Fix

Lower resources.requests

Add node resources

------------------------------------------------------------------------

## Question 28

**Full Question:** Given a pod mounting a PVC that doesn't exist,
diagnose the FailedMount/Pending and create the PVC.

### Diagnose

``` bash
kubectl describe pod <pod>
```

``` bash
kubectl get pvc
```

Example:

FailedMount

PersistentVolumeClaim not found

### Fix

Create the PVC or reference the correct PVC.

------------------------------------------------------------------------

## Question 29

**Full Question:** A container based on ubuntu with no long-running
command shows Completed/CrashLoopBackOff. Explain

why and add a proper command.

### Problem

Ubuntu has no long-running process.

### Fix

sleep infinity

or

tail -f /dev/null

------------------------------------------------------------------------

## Question 30

**Full Question:** Use kubectl get events --sort-by=.lastTimestamp (or
kubectl events) to triage everything that happened to a

pod over time.

``` bash
kubectl get events --sort-by=.lastTimestamp
```

or

``` bash
kubectl events
```

Shows the complete event history for troubleshooting

------------------------------------------------------------------------

## Question 31

**Full Question:** Use kubectl exec -it to get a shell in a pod and
debug DNS with nslookup and cat /etc/resolv.conf.

Enter the Pod:

``` bash
kubectl exec -it <pod> -- sh
```

Check DNS:

nslookup service-name

cat /etc/resolv.conf

------------------------------------------------------------------------

## Question 32

**Full Question:** Use an ephemeral debug container (kubectl debug -it
`<pod>`{=html} --image=busybox --target=`<container>`{=html}) to

troubleshoot a no-shell/distroless container.

``` bash
kubectl debug -it <pod> \
```

--image=busybox\

--target=`<container>`{=html}

Adds a temporary debug container.

------------------------------------------------------------------------

## Question 33

**Full Question:** Spin up a throwaway kubectl run tmp --rm -it
--image=busybox pod to test connectivity to a Service from

inside the cluster.

``` bash
kubectl run tmp \
```

--rm -it\

--image=busybox

Inside:

wget service-name

or

nslookup service-name

------------------------------------------------------------------------

## Question 34

**Full Question:** A node shows NotReady. List what you'd check
(kubelet, disk pressure, CNI) and inspect it on minikube with

``` bash
kubectl describe node and sudo minikube logs.
```

Check

``` bash
kubectl describe node <node>
```

Look for:

kubelet

DiskPressure

MemoryPressure

PIDPressure

Network/CNI

Minikube:

sudo minikube logs

------------------------------------------------------------------------

## Question 35

**Full Question:** A pod is stuck Terminating. Explain finalizers and
the grace period, then force-delete it (--grace-period=0 --

force) and state the risks.

### Cause

Finalizers

Long grace period

Force delete

``` bash
kubectl delete pod <pod> \
```

--grace-period=0\

--force

Risk

Application may not shut down cleanly and data may be lost.

------------------------------------------------------------------------

## Question 36

**Full Question:** Given a manifest with the wrong apiVersion/kind
pairing (e.g. apps/v1 + Pod), explain the validation error

and correct the group/version.

Wrong:

``` bash
apiVersion: apps/v1
```

kind: Pod

Correct:

``` bash
apiVersion: v1
```

kind: Pod

or

``` bash
apiVersion: apps/v1
```

kind: Deployment

------------------------------------------------------------------------

## Question 37

**Full Question:** A pod fails with CreateContainerConfigError because a
configMapKeyRef key is missing. Diagnose and fix.

### Cause

Missing ConfigMap key.

### Diagnose:

``` bash
kubectl describe pod <pod>
```

``` bash
kubectl get configmap
```

### Fix:

Add missing key.

Correct configMapKeyRef.

------------------------------------------------------------------------

## Question 38

**Full Question:** A pod can't mount a Secret that lives in a different
namespace. Explain namespace scoping and fix it.

Secrets are namespace-scoped.

A Pod cannot mount a Secret from another namespace.

### Fix

Create the Secret in the same namespace.

Move the Pod to the correct namespace.

------------------------------------------------------------------------

## Question 39

**Full Question:** A rollout is stuck with ProgressDeadlineExceeded. Use
kubectl describe deployment / rollout status to find

the cause and remediate.

### Diagnose:

``` bash
kubectl rollout status deployment <deployment>
```

``` bash
kubectl describe deployment <deployment>
```

Common causes:

Pods not Ready

ImagePullBackOff

CrashLoopBackOff

### Fix the root cause and redeploy.

------------------------------------------------------------------------

## Question 40

**Full Question:** Catch indentation/field typos (e.g. imagePullpolicy,
misnested ports) before applying with kubectl apply --

dry-run=server / --validate=true, then fix them.

Before applying:

``` bash
kubectl apply \
```

--dry-run=server\

-f deployment.yaml

or

``` bash
kubectl apply \
```

--validate=true\

-f deployment.yaml

Finds:

indentation errors

unknown fields

spelling mistakes

------------------------------------------------------------------------

## Question 41

**Full Question:** An app's logs say it can't reach its database
Service. Verify in order: pod running → Service exists →

endpoints populated → DNS resolves → correct port --- and identify the
broken link.

Check in this order:

``` bash
kubectl get pods
```

↓

``` bash
kubectl get svc
```

↓

``` bash
kubectl get endpoints
```

↓

nslookup database-service

↓

Verify Service port matches container port.

------------------------------------------------------------------------

## Question 42

**Full Question:** Read a container's state/lastState and restartCount
with kubectl get pod -o jsonpath to determine the root

### cause of repeated restarts.

Find Restart Cause

``` bash
kubectl get pod <pod> \
```

-o jsonpath='{.status.containerStatuses\[\*\].restartCount}'

Last state:

``` bash
kubectl get pod <pod> \
```

-o jsonpath='{.status.containerStatuses\[\*\].lastState}'

Useful for identifying:

OOMKilled

Error

CrashLoopBackOff

------------------------------------------------------------------------

## Question 43

**Full Question:** Compare OpenShift Route to Ingress

Route Ingress

OpenShift only Standard Kubernetes

Exposes Service externally Exposes Service externally

Simpler in OpenShift Requires Ingress Controller

Supports TLS Supports TLS

### Summary

Route = OpenShift-specific way to expose a Service.

Ingress = Standard Kubernetes resource for exposing HTTP/HTTPS services.
