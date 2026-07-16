# Kubernetes Pods Deep Dive Lab

Hands-on exploration of Kubernetes Pods — creating them imperatively and declaratively, testing pod-to-pod networking across nodes, debugging with throwaway pods, working with multi-container (sidecar) pods, and understanding pod lifecycle failures like `CrashLoopBackOff`.

---

## Objective

This lab teaches the practical mechanics of how Pods actually work in a Kubernetes cluster — not just how to create one, but how it gets networked, how to debug it when things go wrong, how multiple containers can share a single pod, and the difference between the imperative and declarative ways of managing resources. This is foundational knowledge: almost every higher-level Kubernetes object (Deployments, StatefulSets, Jobs) is ultimately managing Pods underneath.

---

## Skills Practiced

- Kubernetes (Pods, networking, lifecycle, multi-container patterns)
- kubectl (imperative commands, declarative `apply`, debugging commands)
- YAML (manifest authoring, multi-document files)
- Bash (heredocs, `awk`, `tee`, environment variable extraction)
- Networking (pod IPs, port-forwarding, cross-node connectivity)
- Linux (SSH between nodes, shell scripting)

---

## Prerequisites

- A running Kubernetes cluster with at least one worker node (e.g., `worker-1`) reachable via SSH from the control plane
- `kubectl` configured and pointed at the cluster
- Basic familiarity with YAML syntax
- Network access between control-plane and worker nodes (for `ping`/SSH steps)

---

## Task Overview

The lab walks through the full spectrum of working with Pods: spinning one up quickly with `kubectl run`, inspecting its network identity, reaching it from a different node entirely, forwarding a local port into it, and then moving from ad-hoc imperative commands to proper YAML manifests managed declaratively with `kubectl apply`. It also covers a more advanced pattern — multi-container "sidecar" pods — and how to inspect and debug pods that fail to start correctly.

Someone unfamiliar with Kubernetes should come away understanding that a Pod is not just "a container" — it's a shared network and storage sandbox that one or more containers run inside, and that Kubernetes gives you a rich set of tools (`describe`, `logs -c`, `exec -c`) specifically because more than one container can live in that sandbox.

---

## Step-by-Step Walkthrough

### Step 1 — Create a Pod imperatively

```bash
kubectl run nginx --image=nginx
```
---

### Step 2 — Inspect the Pod's network identity

```bash
kubectl get pod -o wide
```
```bash
NGINX_IP=$(kubectl get pods -o wide | awk '/nginx/ {print $6}')
```
---

### Step 3 — Test pod connectivity across nodes

```bash
ping -c 3 $NGINX_IP
ssh worker-1 ping -c 3 $NGINX_IP
```
---

### Step 4 — Port-forward into the Pod

```bash
kubectl port-forward pod/nginx 8080:80
```
---

### Step 5 — Debug with a temporary, one-off Pod

```bash
kubectl run -it --rm curl --image=curlimages/curl:8.4.0 --restart=Never -- http://$NGINX_IP
```

**Flags:**
- `-it` — interactive terminal, so you can see the curl output live.
- `--rm` — deletes the Pod automatically once it exits, avoiding leftover debug pods.
- `--image=curlimages/curl:8.4.0` — uses a minimal, purpose-built curl image (pinned to a specific version, which is good practice for reproducibility).
- `--restart=Never` — tells Kubernetes not to restart this Pod after it completes; without this, Kubernetes would treat the curl command exiting as a "crash" and keep restarting it.
- `--` — separates kubectl's own flags from the command/arguments passed *into* the container. Everything after `--` is passed to the curl binary, not interpreted by kubectl.
  
---

### Step 6 — Generate and inspect a YAML manifest

```bash
kubectl run ubuntu --image=ubuntu --dry-run=client -o yaml -- sleep infinity > ubuntu.yaml
```
**Flags:**
- `--dry-run=client` — builds the object definition and prints it, but never actually sends it to the cluster or creates anything.
- `-o yaml` — outputs the result as YAML instead of the default human-readable table.
- `-- sleep infinity` — again using `--` to pass `sleep infinity` as the container's command, keeping the ubuntu container alive indefinitely instead of exiting immediately (ubuntu's default behavior with no command is to exit).

```bash
kubectl explain pod.spec.restartPolicy
```
---

### Step 7 — Create and apply manifests (imperative vs declarative)

```bash
kubectl create -f nginx.yaml
kubectl apply -f nginx.yaml
```
---

### Step 8 — Combine multiple resources into one file

```bash
{ cat nginx.yaml; echo "---"; cat ubuntu.yaml; } | tee combined.yaml
```

```bash
cat <<EOF > combined.yaml
...
EOF
```

```bash
kubectl apply -f combined.yaml
```
---

### Step 9 — Debug a multi-container (sidecar) Pod

```bash
kubectl describe pod/mypod
kubectl logs pod/mypod -c sidecar
kubectl exec -it pod/mypod -c sidecar -- bash
```
---

### Step 10 — Clean up

```bash
kubectl delete pods nginx ubuntu --now
```
---

## Verification

- `kubectl get pod -o wide` shows the Pod in `Running` state with an assigned IP and node.
- `ping` succeeding (0% packet loss) from both the control plane and `worker-1` confirms cluster-wide Pod network reachability.
- A successful `curl` response from the temporary debug pod confirms the nginx Pod is actually serving HTTP traffic, not just "running."
- `kubectl port-forward` allowing a local browser/curl request to `localhost:8080` to reach nginx confirms the port mapping works.
- `kubectl describe pod/mypod` showing no error Events, and `Ready` containers, confirms a healthy multi-container Pod.
- `kubectl logs pod/mypod -c sidecar` returning expected log output (rather than an error) confirms the `-c` targeting works correctly.

---

## Key Concepts Learned

**Imperative vs Declarative management**
`kubectl run`/`create` tell Kubernetes exactly what to do, once. `kubectl apply` describes the *desired end state* and lets Kubernetes figure out what needs to change to get there — and can be run repeatedly and safely. Production environments almost always use the declarative approach (`apply`, or GitOps tooling built on top of it) because it's idempotent and safe to re-run in automation/CI pipelines.

**Pod networking**
Every Pod gets its own IP from the cluster's CNI network, and that IP is routable from *any* node in the cluster — not just the node the Pod happens to run on. This is what allows Services, DNS, and cross-node communication to work transparently in real clusters.

**Port-forwarding**
A quick, temporary way to reach into a specific Pod for testing without setting up a Service or Ingress — heavily used during development and debugging, but not a substitute for proper Service-based exposure in production.

**Temporary debug pods**
Spinning up a throwaway Pod (`--rm --restart=Never`) with a tool like `curl` is one of the most common real-world Kubernetes debugging techniques — it lets you test connectivity *from inside* the cluster's network, which behaves differently than testing from your own laptop.

**Multi-container Pods (sidecar pattern)**
Containers in the same Pod share network and can share volumes, making the sidecar pattern useful for logging agents, proxies, or auxiliary processes that support a main application container. This is different from **init containers**, which run to completion sequentially *before* the main containers start — sidecars run *alongside* the main container for its whole lifetime.

**Restart policy and CrashLoopBackOff**
The default `restartPolicy: Always` means Kubernetes keeps the Pod's "sandbox" (its IP, its identity) alive and just restarts failing containers inside it. Repeated failures trigger `CrashLoopBackOff`, where Kubernetes progressively increases the delay between restart attempts rather than restart-looping as fast as possible — this is a deliberate design to avoid hammering a failing workload.

---

## Takeaways

- Test pod connectivity with a lightweight, disposable `curl` pod — it's a pattern worth reusing constantly.
- `kubectl describe` is the first stop for diagnosing any stuck or failing Pod.
- The `--` separator is easy to forget but critical whenever passing arguments through to a container's command.
- `kubectl apply` is the production-recommended approach because of its idempotency and safe re-runnability.
- Sidecars and init containers solve different problems: "alongside" vs. "before."

---

## Repository Structure

```
labs/
└── pods/
    ├── README.md
    ├── nginx.yaml
    ├── ubuntu.yaml
    └── combined.yaml
```
---

## Reflection

This lab really solidified for me that a Pod isn't just "a container" — it's a network sandbox that one or more containers share, which is why sidecars can talk over `localhost` and why `kubectl logs`/`exec` need a `-c` flag once there's more than one container involved. Testing pod connectivity from a completely different node than the one running the Pod made the "every Pod gets a cluster-wide routable IP" concept concrete rather than theoretical. I also came away with a much clearer mental model of `create` vs `apply` — and why almost everyone recommends `apply` for anything beyond a quick one-off test. Small syntax mistakes (a missing `--`, a missing `=`) were some of the most instructive moments, since fixing them forced me to understand exactly what kubectl was doing with those flags in the first place.

---

## Future Improvements

- Add an init container that waits for a service to be available before starting nginx.
- Set up readiness and liveness probes for the sidecar.
- Extend the sidecar to act as a simple monitoring agent that logs HTTP response codes.
