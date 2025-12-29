# Lab 3: Controllers, Deployments, and ReplicaSets

**Instructor Guide**

This lab develops deep understanding of Kubernetes controllers, specifically Deployments and ReplicaSets. You will learn what happens behind the scenes for every action so you can confidently explain and debug student issues.

---

## 1. Mental Model: Why Controllers Exist

### The Problem with Bare Pods

In Lab 2, you created Pods directly. You learned that:
- **kubelet** watches Pods assigned to its node
- **kubelet** restarts containers if they crash
- **kubelet** does NOT recreate Pods if they are deleted

This creates a fundamental problem: if a Pod is deleted (node failure, eviction, manual deletion), it's gone forever. No component in Kubernetes will recreate it.

### The Controller Pattern

Kubernetes solves this with **controllers**—control loops that continuously reconcile desired state with actual state.

**Key insight:** Controllers don't "do things to your cluster." They observe what exists, compare it to what should exist, and take action to close the gap.

This is called a **reconciliation loop**:

```
1. Read desired state (from etcd via API server)
2. Read actual state (from API server)
3. Compare
4. If different, take action to make actual match desired
5. Wait, then repeat
```

### Why Kubernetes Prefers Replacement Over Mutation

When you change a Deployment's container image, Kubernetes doesn't modify existing Pods. It creates new Pods and deletes old ones.

**Why?**
- **Immutability:** Pods are designed to be disposable
- **Predictability:** You know exactly what you're getting (no partial updates)
- **Rollback:** Old state is preserved in old ReplicaSets
- **Safety:** Failed updates don't corrupt running Pods

This is not a limitation. This is a design choice that makes Kubernetes reliable.

---

## 2. Creating a Deployment

### Understanding the Object Hierarchy

A **Deployment** is a high-level abstraction that manages:
- **ReplicaSets** (which manage Pods)
- **Rolling updates** (creating new ReplicaSets, scaling down old ones)
- **Rollback history** (keeping old ReplicaSets around)

The ownership chain:
```
Deployment → ReplicaSet → Pod
```

Each arrow represents an **ownerReference** in the child object's metadata.

### Create a Deployment YAML

Create a file called `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Explanation of Every Major Field

**`apiVersion: apps/v1`**
- Tells the API server which API group and version to use
- `apps/v1` is the stable API for Deployments
- The API server uses this to validate the schema

**`kind: Deployment`**
- Tells the API server what type of object this is
- The **Deployment controller** watches for objects of this kind

**`metadata.name`**
- Unique identifier for this Deployment in its namespace
- Used in `kubectl` commands and in ownerReferences

**`metadata.labels`**
- Key-value pairs for organizing objects
- NOT used for Pod selection (that's `spec.selector`)
- Used for querying: `kubectl get deployments -l app=nginx`

**`spec.replicas: 3`**
- **Desired state:** "I want 3 Pods"
- The **ReplicaSet controller** reads this field
- This is what the controller continuously reconciles

**`spec.selector.matchLabels`**
- **Critical:** Tells the ReplicaSet which Pods it owns
- The ReplicaSet controller queries the API server: "show me all Pods with label `app=nginx`"
- **Must match** `spec.template.metadata.labels` or the API server rejects the Deployment

**`spec.template`**
- This is a **Pod template**
- When the ReplicaSet needs to create a Pod, it uses this template
- `template.metadata.labels` must match `spec.selector.matchLabels`

**`spec.template.spec.containers`**
- The actual container specification
- **kubelet** reads this when running the Pod
- The Deployment controller never touches this directly

### Apply the Deployment

```bash
kubectl apply -f nginx-deployment.yaml
```

**What happens:**

1. **kubectl** sends the YAML to the **API server**
2. **API server** validates the schema, stores it in **etcd**
3. **Deployment controller** (running in kube-controller-manager) sees the new Deployment
4. **Deployment controller** creates a **ReplicaSet** with:
   - `replicas: 3`
   - `selector: app=nginx`
   - `template: <the Pod template>`
5. **ReplicaSet controller** sees the new ReplicaSet
6. **ReplicaSet controller** queries: "how many Pods with label `app=nginx` exist?" (answer: 0)
7. **ReplicaSet controller** creates 3 Pods using the template
8. **Scheduler** assigns each Pod to a node
9. **kubelet** on each node sees Pods assigned to it, pulls the image, starts containers

**All of this happens in seconds.**

---

## 3. Observing the Ownership Chain

### View the Deployment

```bash
kubectl get deployments
```

**Output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s
```

**Fields explained:**
- **READY:** 3/3 means "3 Pods are ready out of 3 desired"
- **UP-TO-DATE:** 3 Pods match the current Pod template
- **AVAILABLE:** 3 Pods are ready and available to serve traffic
- **AGE:** Time since Deployment was created

**Who updates these fields?** The **Deployment controller** reads ReplicaSet status and aggregates it.

### View the ReplicaSet

```bash
kubectl get replicasets
```

**Output:**
```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7d64c5f5d9   3         3         3       30s
```

**Notice:**
- The ReplicaSet name is `nginx-deployment-<hash>`
- The hash (`7d64c5f5d9`) is computed from the **Pod template**
- If you change the Pod template, you get a **new ReplicaSet** with a different hash

**Who created this ReplicaSet?** The **Deployment controller**.

**Who updates DESIRED/CURRENT/READY?** The **ReplicaSet controller**.

### View the Pods

```bash
kubectl get pods
```

**Output:**
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7d64c5f5d9-abc12   1/1     Running   0          30s
nginx-deployment-7d64c5f5d9-def34   1/1     Running   0          30s
nginx-deployment-7d64c5f5d9-ghi56   1/1     Running   0          30s
```

**Notice:**
- Pod names are `<replicaset-name>-<random-suffix>`
- All Pods have the same prefix because they're owned by the same ReplicaSet

**Who created these Pods?** The **ReplicaSet controller**.

**Who runs the containers?** The **kubelet** on each node.

### Verify Ownership

```bash
kubectl get pods -o yaml | grep -A 5 ownerReferences
```

**You'll see:**
```yaml
ownerReferences:
- apiVersion: apps/v1
  kind: ReplicaSet
  name: nginx-deployment-7d64c5f5d9
  uid: <some-uid>
  controller: true
```

**This means:**
- The Pod is owned by the ReplicaSet
- If the ReplicaSet is deleted, the Pod is deleted (garbage collection)
- The ReplicaSet controller is responsible for this Pod's lifecycle

**Check the ReplicaSet's owner:**

```bash
kubectl get replicaset nginx-deployment-7d64c5f5d9 -o yaml | grep -A 5 ownerReferences
```

**You'll see:**
```yaml
ownerReferences:
- apiVersion: apps/v1
  kind: Deployment
  name: nginx-deployment
  uid: <some-uid>
  controller: true
```

**The chain is complete:**
```
Deployment (you created)
  ↓ owns
ReplicaSet (Deployment controller created)
  ↓ owns
Pods (ReplicaSet controller created)
```

---

## 4. Self-Healing in Action

### Delete a Pod Manually

Pick one Pod name from `kubectl get pods`:

```bash
kubectl delete pod nginx-deployment-7d64c5f5d9-abc12
```

**Immediately run:**

```bash
kubectl get pods -w
```

**What you'll see:**
```
NAME                                READY   STATUS        RESTARTS   AGE
nginx-deployment-7d64c5f5d9-abc12   1/1     Terminating   0          2m
nginx-deployment-7d64c5f5d9-xyz99   0/1     Pending       0          0s
nginx-deployment-7d64c5f5d9-xyz99   0/1     ContainerCreating   0     0s
nginx-deployment-7d64c5f5d9-xyz99   1/1     Running       0          2s
```

### What Just Happened?

**Step-by-step:**

1. **You** sent a DELETE request to the API server
2. **API server** marked the Pod for deletion (added `deletionTimestamp`)
3. **kubelet** saw the deletion timestamp, stopped the container, removed the Pod
4. **ReplicaSet controller** (running its reconciliation loop) noticed:
   - Desired: 3 Pods
   - Actual: 2 Pods (one was deleted)
   - **Gap detected**
5. **ReplicaSet controller** created a new Pod using the template
6. **Scheduler** assigned it to a node
7. **kubelet** started the container

**Critical insight:** The **Deployment** itself did not change. The **ReplicaSet** did not change. Only the **Pod count** changed, and the **ReplicaSet controller** reconciled it.

### Verify the Deployment is Unchanged

```bash
kubectl get deployment nginx-deployment -o yaml | grep -A 2 "generation:"
```

**You'll see:**
```yaml
generation: 1
```

**`generation`** is incremented only when you modify `spec`. Deleting a Pod doesn't modify the Deployment's spec, so generation stays at 1.

**This proves:** The Deployment controller didn't react to the Pod deletion. The ReplicaSet controller did.

---

## 5. Rolling Update Mechanics

### Change the Container Image

Edit the Deployment to use a newer nginx version:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
```

**Alternative (more explicit):**

```bash
kubectl edit deployment nginx-deployment
```

Change `image: nginx:1.21` to `image: nginx:1.22`, save, and exit.

### Watch What Happens

**In one terminal:**

```bash
kubectl get pods -w
```

**In another terminal:**

```bash
kubectl get replicasets -w
```

**What you'll observe:**

**ReplicaSets:**
```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7d64c5f5d9   3         3         3       5m
nginx-deployment-6c8b7f9d8f   1         0         0       0s
nginx-deployment-6c8b7f9d8f   1         1         0       0s
nginx-deployment-6c8b7f9d8f   1         1         1       2s
nginx-deployment-7d64c5f5d9   2         3         3       5m
nginx-deployment-6c8b7f9d8f   2         1         1       2s
...
nginx-deployment-6c8b7f9d8f   3         3         3       10s
nginx-deployment-7d64c5f5d9   0         0         0       5m
```

**Pods:**
```
nginx-deployment-6c8b7f9d8f-new1   0/1   Pending             0     0s
nginx-deployment-6c8b7f9d8f-new1   0/1   ContainerCreating   0     0s
nginx-deployment-6c8b7f9d8f-new1   1/1   Running             0     2s
nginx-deployment-7d64c5f5d9-abc12  1/1   Terminating         0     5m
...
```

### What Just Happened? (Detailed Breakdown)

**1. You changed the Deployment's Pod template**
- The API server incremented `metadata.generation` (now 2)
- The Deployment controller noticed the change

**2. Deployment controller created a NEW ReplicaSet**
- Name: `nginx-deployment-6c8b7f9d8f` (different hash because template changed)
- Initial replicas: 0
- Template: nginx:1.22

**3. Deployment controller orchestrates the rollout**
- Strategy: `RollingUpdate` (default)
- Parameters:
  - `maxUnavailable: 25%` (at most 1 Pod can be unavailable during update)
  - `maxSurge: 25%` (at most 1 extra Pod can exist during update)

**4. Deployment controller scales up new ReplicaSet**
- Sets `nginx-deployment-6c8b7f9d8f.spec.replicas = 1`
- ReplicaSet controller creates 1 new Pod

**5. Deployment controller waits for new Pod to be Ready**
- Polls the ReplicaSet status
- Sees `READY: 1`

**6. Deployment controller scales down old ReplicaSet**
- Sets `nginx-deployment-7d64c5f5d9.spec.replicas = 2`
- ReplicaSet controller deletes 1 old Pod

**7. Repeat steps 4-6 until:**
- New ReplicaSet has 3 Pods
- Old ReplicaSet has 0 Pods

**8. Old ReplicaSet is NOT deleted**
- It's kept for rollback history
- Default: last 10 ReplicaSets are retained

### Why Two ReplicaSets Coexist

**During the rollout:**
- Old ReplicaSet: manages old Pods (nginx:1.21)
- New ReplicaSet: manages new Pods (nginx:1.22)

**This allows:**
- **Gradual transition:** Not all Pods change at once
- **Availability:** Some Pods are always running
- **Rollback:** If new Pods fail, old Pods are still there

**After the rollout:**
- Old ReplicaSet: `replicas: 0` (kept for history)
- New ReplicaSet: `replicas: 3` (active)

### Why Old Pods Are Not Modified

**Kubernetes does not:**
- Stop a container
- Pull a new image
- Restart the container with the new image

**Kubernetes does:**
- Create new Pods with the new image
- Delete old Pods with the old image

**Why?**
- **Pod spec is immutable** (most fields cannot be changed after creation)
- **Containers are immutable** (you can't change a running container's image)
- **Predictability:** You know exactly what's running (no partial updates)

**This is not a workaround. This is the design.**

---

## 6. Rollback Behavior

### View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

**Output:**
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

**Explanation:**
- Revision 1: nginx:1.21
- Revision 2: nginx:1.22
- CHANGE-CAUSE is empty because we didn't annotate the Deployment

**To add CHANGE-CAUSE in the future:**

```bash
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Updated to nginx:1.22"
```

### View a Specific Revision

```bash
kubectl rollout history deployment/nginx-deployment --revision=1
```

**Output shows the Pod template from revision 1 (nginx:1.21).**

### Rollback to Previous Revision

```bash
kubectl rollout undo deployment/nginx-deployment
```

**Watch the rollout:**

```bash
kubectl get replicasets -w
```

**What you'll see:**
```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-6c8b7f9d8f   3         3         3       5m
nginx-deployment-7d64c5f5d9   0         0         0       10m
nginx-deployment-7d64c5f5d9   1         0         0       10m
nginx-deployment-7d64c5f5d9   1         1         0       10m
nginx-deployment-6c8b7f9d8f   2         3         3       5m
...
nginx-deployment-7d64c5f5d9   3         3         3       10m
nginx-deployment-6c8b7f9d8f   0         0         0       5m
```

### What Just Happened?

**1. You ran `kubectl rollout undo`**
- kubectl queried the Deployment's rollout history
- Found the previous ReplicaSet: `nginx-deployment-7d64c5f5d9`

**2. Deployment controller updated the Deployment's Pod template**
- Changed `image: nginx:1.22` back to `image: nginx:1.21`
- Incremented `metadata.generation` (now 3)

**3. Deployment controller noticed the template matches an old ReplicaSet**
- Instead of creating a NEW ReplicaSet, it reused `nginx-deployment-7d64c5f5d9`
- This is why the hash stayed the same

**4. Deployment controller orchestrated the rollout**
- Scaled up old ReplicaSet (7d64c5f5d9) from 0 to 3
- Scaled down new ReplicaSet (6c8b7f9d8f) from 3 to 0

**5. Result:**
- Old ReplicaSet is now active again
- New ReplicaSet is kept for history (in case you want to roll forward)

### What Kubernetes Remembers

**Kubernetes does NOT store:**
- A "snapshot" of your cluster
- A "backup" of your Pods

**Kubernetes DOES store:**
- Old ReplicaSets (with `replicas: 0`)
- Each ReplicaSet has the Pod template from that revision

**Rollback is just:**
- Updating the Deployment's Pod template to match an old ReplicaSet
- Scaling up that old ReplicaSet
- Scaling down the current ReplicaSet

**This is why rollback is fast:** The old ReplicaSet already exists. No need to reconstruct history.

### Rollback to a Specific Revision

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

**This explicitly rolls back to revision 1, even if it's not the immediate previous revision.**

---

## 7. Debugging Mindset

### Common Failure Scenarios

#### Scenario 1: Image Pull Failure

**Simulate it:**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:nonexistent
```

**Observe:**

```bash
kubectl get pods
```

**Output:**
```
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-abc123-xyz99       0/1     ImagePullBackOff   0          2m
nginx-deployment-7d64c5f5d9-old1    1/1     Running            0          10m
nginx-deployment-7d64c5f5d9-old2    1/1     Running            0          10m
```

**What happened:**
- Deployment controller created a new ReplicaSet
- ReplicaSet controller created new Pods
- **kubelet** tried to pull `nginx:nonexistent`, failed
- **kubelet** retries with exponential backoff (ImagePullBackOff)
- Deployment controller sees new Pods are not Ready
- Deployment controller **stops the rollout** (does not scale down old Pods)

**Why old Pods are still running:**
- `maxUnavailable: 25%` means at least 2 Pods must be available
- New Pods are not Ready, so old Pods must stay

**Where to look:**

```bash
kubectl describe pod nginx-deployment-abc123-xyz99
```

**Look for:**
- **Events:** "Failed to pull image"
- **State:** "Waiting: ImagePullBackOff"

**Fix it:**

```bash
kubectl rollout undo deployment/nginx-deployment
```

#### Scenario 2: Selector Mismatch

**Simulate it:**

Create a Deployment with mismatched selector:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: apache  # MISMATCH!
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**Try to apply:**

```bash
kubectl apply -f broken-deployment.yaml
```

**Output:**
```
The Deployment "broken-deployment" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"app":"apache"}: `selector` does not match template `labels`
```

**What happened:**
- **API server** validated the Deployment
- **API server** rejected it before storing in etcd
- No controller ever saw this object

**Lesson:** The API server enforces schema validation. Some errors are caught immediately.

#### Scenario 3: Stuck Rollout

**Simulate it:**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.23
```

**Assume the new image has a bug that causes the container to crash immediately.**

**Observe:**

```bash
kubectl rollout status deployment/nginx-deployment
```

**Output:**
```
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
```

**Check Pods:**

```bash
kubectl get pods
```

**Output:**
```
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-new123-xyz99       0/1     CrashLoopBackOff   5          5m
nginx-deployment-7d64c5f5d9-old1    1/1     Running            0          20m
nginx-deployment-7d64c5f5d9-old2    1/1     Running            0          20m
```

**What happened:**
- New Pod keeps crashing
- **kubelet** restarts it (CrashLoopBackOff)
- Deployment controller waits for it to be Ready
- It never becomes Ready
- Deployment controller does not proceed (old Pods stay)

**Where to look:**

```bash
kubectl logs nginx-deployment-new123-xyz99
kubectl describe pod nginx-deployment-new123-xyz99
```

**Fix it:**

```bash
kubectl rollout undo deployment/nginx-deployment
```

**Or set a deadline:**

Deployments have a `progressDeadlineSeconds` field (default: 600 seconds). If a rollout doesn't progress within this time, it's marked as failed.

```bash
kubectl describe deployment nginx-deployment
```

**Look for:**
```
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    False   ProgressDeadlineExceeded
```

### Where to Look First

**For Pod issues:**
1. `kubectl get pods` → Check STATUS
2. `kubectl describe pod <name>` → Check Events
3. `kubectl logs <name>` → Check application logs

**For Deployment issues:**
1. `kubectl get deployment` → Check READY, UP-TO-DATE, AVAILABLE
2. `kubectl describe deployment <name>` → Check Conditions, Events
3. `kubectl rollout status deployment/<name>` → Check rollout progress

**For ReplicaSet issues:**
1. `kubectl get replicasets` → Check DESIRED vs CURRENT vs READY
2. `kubectl describe replicaset <name>` → Check Events

**Control flow:**
```
Deployment (high-level intent)
  ↓
ReplicaSet (desired Pod count)
  ↓
Pod (actual workload)
  ↓
Container (running process)
```

**Debug from the bottom up:**
- Container logs → Pod status → ReplicaSet status → Deployment status

---

## 8. Mandatory Mental Checkpoint

Before moving to the next lab, you must be able to answer these questions confidently:

### Question 1: Who creates Pods?

**Answer:** The **ReplicaSet controller** creates Pods.

**Not:**
- The Deployment controller (it creates ReplicaSets)
- The kubelet (it runs Pods, doesn't create them)
- The Scheduler (it assigns Pods to nodes, doesn't create them)

### Question 2: Who restarts containers?

**Answer:** The **kubelet** restarts containers.

**Not:**
- The Deployment controller (it doesn't interact with containers)
- The ReplicaSet controller (it doesn't interact with containers)
- The container runtime (it's instructed by kubelet)

### Question 3: Who decides desired vs actual state?

**Answer:**
- **You** decide desired state (by writing YAML)
- **Controllers** compare desired vs actual and take action
- **API server** stores desired state (in etcd)
- **Controllers** read actual state (from API server)

**Specifically:**
- Deployment controller: desired Deployment spec vs actual ReplicaSet state
- ReplicaSet controller: desired replica count vs actual Pod count
- kubelet: desired Pod spec vs actual container state

### Question 4: Why is deleting Pods meaningless under a Deployment?

**Answer:** Because the **ReplicaSet controller** will immediately recreate them.

**The reconciliation loop:**
1. ReplicaSet says: "I want 3 Pods with label `app=nginx`"
2. You delete a Pod
3. ReplicaSet controller queries: "How many Pods with label `app=nginx` exist?" (answer: 2)
4. ReplicaSet controller: "I want 3, I have 2, I need to create 1"
5. ReplicaSet controller creates a new Pod

**This is not a bug. This is the entire point of controllers.**

### Question 5: What's the difference between kubelet and controller responsibility?

**kubelet:**
- Watches Pods assigned to its node
- Pulls images
- Starts containers
- Restarts containers if they crash
- Reports Pod status to API server
- **Does NOT create or delete Pods**

**ReplicaSet controller:**
- Watches ReplicaSets
- Creates Pods to match desired replica count
- Deletes Pods if there are too many
- **Does NOT run containers**

**Deployment controller:**
- Watches Deployments
- Creates/updates ReplicaSets
- Orchestrates rolling updates
- **Does NOT create Pods directly**

**Separation of concerns:**
- Controllers manage **desired state** (declarative)
- kubelet manages **actual state** (imperative execution)

---

## Conclusion

**Kubernetes is not restarting Pods for you.**

**Controllers are continuously reconciling desired state.**

**This is why you almost never create Pods directly in production.**

---

### What You Now Understand

1. **Controllers are reconciliation loops**, not event handlers
2. **Deployments manage ReplicaSets**, ReplicaSets manage Pods
3. **Self-healing is continuous reconciliation**, not magic
4. **Rolling updates create new ReplicaSets**, they don't modify Pods
5. **Rollbacks reuse old ReplicaSets**, they don't restore snapshots
6. **Immutability is a design choice**, not a limitation

### What You Can Now Explain

- Why deleting a Pod under a Deployment is meaningless
- Why changing a Deployment's image creates new Pods instead of updating old ones
- Why two ReplicaSets coexist during a rollout
- Why rollback is fast (old ReplicaSets are kept)
- Where to look when a rollout is stuck
- The difference between kubelet responsibility and controller responsibility

### What You Can Now Debug

- Image pull failures (check Pod events)
- Selector mismatches (API server rejects)
- Stuck rollouts (check Pod status, logs)
- CrashLoopBackOff (check container logs)
- Pods not being created (check ReplicaSet events)
- Rollouts not progressing (check Deployment conditions)

---

**You are now ready to teach this material with confidence.**

When a student asks "why did my Pod restart?", you can calmly explain:
- If the **container** crashed, kubelet restarted it
- If the **Pod** was deleted, the ReplicaSet controller recreated it
- If the **image** changed, the Deployment controller created a new Pod

**You understand the mechanics. You can explain every step. You can debug any issue.**

This is the foundation of Kubernetes operations.
