# Lab 05: Services — Connecting Pods

**Goal**: Understand what a Service is, how it finds and connects Pods, and how to use it to build a multi-Pod application.

**Prerequisites**: Ability to write basic Pod YAMLs (see Lab 04)

**Duration**: 40-50 minutes

---

## Stage 1: Guided — Your First Service

### What is a Service?

A **Service** is a stable network endpoint that routes traffic to Pods.

**The Problem**: Pods have IP addresses, but those IPs change when Pods restart. You can't hardcode Pod IPs in your application.

**The Solution**: A Service provides a stable DNS name and IP that doesn't change. It automatically finds the right Pods and routes traffic to them.

**Key Insight**: A Service does NOT run containers. It's a routing rule.

---

### Creating a Database Pod (with Labels)

First, create a database Pod. Notice the **labels** in metadata — this is how Services will find it.

Create `postgres-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  labels:
    app: postgres
    tier: database
spec:
  containers:
  - name: postgres
    image: postgres:15
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_USER
      value: "appuser"
    - name: POSTGRES_PASSWORD
      value: "apppassword"
    - name: POSTGRES_DB
      value: "appdb"
```

**Understanding the Labels Section**:

- **`labels:`** - Key-value pairs attached to the Pod for identification and organization
  - **`app: postgres`** - Identifies this Pod as part of the "postgres" application (used by Services to select this Pod)
  - **`tier: database`** - Categorizes this Pod as belonging to the database tier (useful for organizing multi-tier applications)

Labels are crucial because Services use **selectors** to find which Pods to route traffic to. Any Pod with matching labels will receive traffic from the Service.

**Apply it**:

```bash
kubectl apply -f postgres-pod.yaml
```

**Verify it's running**:

```bash
kubectl get pod postgres-pod
```

---

### Creating a Service for the Database

Now create a Service that will route traffic to this Pod.

Create `postgres-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
```

### Understanding Each Section

#### `kind: Service`
- **What it does**: Declares you're creating a Service resource
- **Why it's needed**: Tells Kubernetes this is a routing rule, not a Pod or other resource

#### `metadata.name: postgres-service`
- **What it does**: Names your Service
- **Why it's critical**: This name becomes a **DNS entry**. Other Pods will use this name to connect
- **Example**: An app can connect to `postgres-service:5432` instead of a Pod IP

#### `spec.selector`
- **What it does**: Defines which Pods this Service should route traffic to
- **How it works**: Kubernetes looks for Pods with **matching labels**
- **In this example**: Any Pod with label `app: postgres` will receive traffic from this Service
- **Think of it as**: A filter that says "send traffic to Pods with these labels"

#### `spec.ports`
- **What it does**: Defines port mapping for the Service
- **Key fields**:
  - `port`: The port the Service listens on (what clients connect to)
  - `targetPort`: The port on the Pod container (where traffic is sent)
  - `protocol`: TCP or UDP

**Mental Model**:
```
Client → Service (port: 5432) → Pod (targetPort: 5432)
```

---

### Apply and Observe

**Step 1**: Create the Service

```bash
kubectl apply -f postgres-service.yaml
```

**Step 2**: List Services

```bash
kubectl get svc
```

You should see:
```
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
postgres-service    ClusterIP   10.96.123.45    <none>        5432/TCP   5s
```

**What to notice**:
- **TYPE**: `ClusterIP` — this Service is only accessible inside the cluster
- **CLUSTER-IP**: A stable internal IP assigned by Kubernetes
- **PORT(S)**: `5432/TCP` — the Service listens on port 5432

**Step 3**: Describe the Service

```bash
kubectl describe svc postgres-service
```

### What to Notice in `describe` Output

Look for these key sections:

**Selector**:
```
Selector:          app=postgres
```
This shows which labels the Service is looking for.

**IP**:
```
IP:                10.96.123.45
```
This is the stable ClusterIP. It won't change even if Pods restart.

**Endpoints**:
```
Endpoints:         172.17.0.5:5432
```
This shows the actual Pod IP(s) that match the selector. This is where traffic actually goes.

**Key Insight**: The Service found your Pod automatically because the Pod has the label `app: postgres`.

---

### Testing the Connection

Let's verify the Service works by checking the endpoints:

```bash
kubectl get endpoints postgres-service
```

You should see:
```
NAME                ENDPOINTS          AGE
postgres-service    172.17.0.5:5432    2m
```

**What this means**: The Service successfully found the Pod and knows where to send traffic.

---

### What Happens If You Delete the Pod?

Try this experiment:

```bash
# Delete the Pod
kubectl delete pod postgres-pod

# Check endpoints immediately
kubectl get endpoints postgres-service

# Recreate the Pod
kubectl apply -f postgres-pod.yaml

# Wait a few seconds, then check endpoints again
kubectl get endpoints postgres-service
```

**What you'll observe**: The Service automatically updates its endpoints when Pods come and go. The Service IP stays the same, but the endpoint (Pod IP) changes.

**This is why Services exist**: Stable routing despite Pod changes.

---

## Understanding Service Types

### Why Different Service Types?

So far, we've used the default Service type: **ClusterIP**. But what if you need to access your application from outside the cluster?

Kubernetes provides different Service types to solve different access problems.

---

### The Three Main Service Types

#### 1. ClusterIP (Default)

**What problem it solves**: Internal communication between Pods

**Who can access it**: Only Pods inside the cluster

**When to use it**:
- Database services (you don't want databases exposed externally)
- Internal APIs
- Microservices talking to each other

**Example**: The `postgres-service` you just created is ClusterIP — only accessible from within the cluster.

---

#### 2. NodePort

**What problem it solves**: Access from outside the cluster using any Node's IP

**Who can access it**: Anyone who can reach the Kubernetes Nodes (external clients)

**When to use it**:
- Development and testing
- Small deployments without a load balancer
- When you need simple external access

**How it works**: Kubernetes opens a specific port (30000-32767) on every Node. Traffic to `NodeIP:NodePort` gets routed to your Service.

**Mental Model**:
```
External Client → Node IP:30080 → Service → Pod
```

---

#### 3. LoadBalancer

**What problem it solves**: Production-grade external access with a dedicated IP

**Who can access it**: Anyone on the internet (or your network)

**When to use it**:
- Production applications
- When you need a stable external IP
- Cloud environments (AWS, GCP, Azure)

**How it works**: Kubernetes asks your cloud provider to create a load balancer. The cloud provider assigns an external IP that routes traffic to your Service.

**Mental Model**:
```
Internet → Cloud Load Balancer (External IP) → Service → Pods
```

**Note**: LoadBalancer only works in cloud environments or with special on-premises setups.

---

### Quick Comparison

| Type | Accessible From | Use Case | External IP |
|------|----------------|----------|-------------|
| **ClusterIP** | Inside cluster only | Internal services, databases | No |
| **NodePort** | Outside cluster via Node IP | Development, testing | No (uses Node IP) |
| **LoadBalancer** | Outside cluster via dedicated IP | Production apps | Yes |

**Note**: There's also **Ingress** (not a Service type, but worth mentioning) — it provides HTTP/HTTPS routing with domain names, typically used for web applications.

---

### Where Service Type is Defined

The Service type is specified in the YAML under `spec.type`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP  # or NodePort, or LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**Key Points**:
- If you don't specify `type`, it defaults to `ClusterIP`
- Changing the type changes **who can access** the Service
- Changing the type does **NOT** change Pod behavior — Pods don't know or care about Service types

---

### Hands-on: Converting to NodePort

Let's modify the `postgres-service` to use NodePort and observe the difference.

**Step 1**: Update `postgres-service.yaml`

Edit the file to add `type: NodePort`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: NodePort  # Add this line
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
```

**Step 2**: Apply the change

```bash
kubectl apply -f postgres-service.yaml
```

**Step 3**: Observe the difference

```bash
kubectl get svc postgres-service
```

You should now see:
```
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
postgres-service    NodePort   10.96.123.45    <none>        5432:30123/TCP   5m
```

**What changed**:
- **TYPE**: Now shows `NodePort` instead of `ClusterIP`
- **PORT(S)**: Now shows `5432:30123/TCP` — the second number (30123) is the NodePort

**What did NOT change**:
- **ClusterIP**: Still `10.96.123.45` — internal access still works the same way
- **Selector**: Still finds the same Pods
- **Endpoints**: Still points to the same Pod IPs
- **Pod behavior**: The Pod doesn't know anything changed

**Step 4**: Describe the Service

```bash
kubectl describe svc postgres-service
```

Look for:
```
Type:              NodePort
NodePort:          <unset>  30123/TCP
```

---

### What This Means

**Before (ClusterIP)**:
- Only Pods inside the cluster could connect to `postgres-service:5432`

**After (NodePort)**:
- Pods inside the cluster can still connect to `postgres-service:5432` (nothing changed for them)
- External clients can now connect to `<any-node-ip>:30123`

**Key Insight**: Service types define **connectivity scope**, not Pod functionality.

---

### Mental Model Reinforcement

| Component | Responsibility |
|-----------|----------------|
| **Pod** | Runs the application/database |
| **Service** | Defines connectivity (how to reach Pods) |
| **Service Type** | Defines who can reach the Service |

**Remember**:
- Pods don't care about Services
- Services don't care about Service types (they just route traffic)
- Service types only affect **accessibility**

---

### Revert to ClusterIP (Recommended)

For the rest of this lab, let's revert to ClusterIP since we're focusing on internal Pod-to-Pod communication:

```bash
# Edit the file and remove the type: NodePort line
# Or explicitly set it back:
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f postgres-service.yaml
```

---

## Stage 2: Semi-Guided — Application → Service

Now let's create an application Pod that connects to the database **using the Service**.

### The Application Pod

Create `app-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo Connecting to postgres-service:5432 && sleep 3600']
    env:
    - name: DATABASE_HOST
      value: "postgres-service"
    - name: DATABASE_PORT
      value: "5432"
    - name: DATABASE_USER
      value: "appuser"
    - name: DATABASE_PASSWORD
      value: "apppassword"
```

### Understanding the Connection

Look at the `env` section:

```yaml
- name: DATABASE_HOST
  value: "postgres-service"
```

**Key Points**:
1. We use the **Service name** (`postgres-service`), NOT a Pod IP
2. Kubernetes provides built-in DNS — any Pod can resolve Service names
3. The Service handles routing to the actual Pod

**Why not use the Pod IP directly?**
- Pod IPs change when Pods restart
- You'd have to update your app configuration every time
- Services provide stability

---

### Apply and Test

**Step 1**: Create the app Pod

```bash
kubectl apply -f app-pod.yaml
```

**Step 2**: Check the app's environment variables

```bash
kubectl exec app-pod -- env | grep DATABASE
```

You should see:
```
DATABASE_HOST=postgres-service
DATABASE_PORT=5432
DATABASE_USER=appuser
DATABASE_PASSWORD=apppassword
```

**Step 3**: Test DNS resolution

```bash
kubectl exec app-pod -- nslookup postgres-service
```

You should see the Service's ClusterIP resolved from the name.

**Step 4**: Test connectivity (optional, if you have network tools)

```bash
kubectl exec app-pod -- nc -zv postgres-service 5432
```

---

### Responsibility Boundaries

Let's be clear about what owns what:

| Responsibility | Owned By |
|----------------|----------|
| Running the database container | **Pod** |
| Database configuration (env vars, image) | **Pod** |
| Stable network endpoint | **Service** |
| Finding Pods using labels | **Service** |
| Routing traffic to the right Pod | **Service** |
| DNS name (`postgres-service`) | **Service** |

**Mental Model**:
- **Pod**: "I run the actual workload"
- **Service**: "I provide a stable way to reach that workload"

---

## Stage 3: Independent Exercise

Now build a complete system from scratch.

### Task

Create a simple web application that connects to a database using a Service.

**Requirements**:

1. **Database Pod**:
   - Name: `mysql-pod`
   - Image: `mysql:8.0`
   - Labels: `app: mysql`, `tier: database`
   - Environment variables:
     - `MYSQL_ROOT_PASSWORD`: `rootpass`
     - `MYSQL_DATABASE`: `myapp`
     - `MYSQL_USER`: `appuser`
     - `MYSQL_PASSWORD`: `apppass`
   - Port: `3306`

2. **Database Service**:
   - Name: `mysql-service`
   - Selector: Must match the database Pod labels
   - Port: `3306` → `3306`

3. **Application Pod**:
   - Name: `webapp-pod`
   - Image: `nginx:1.25` (we'll use nginx as a placeholder app)
   - Labels: `app: webapp`, `tier: frontend`
   - Environment variables:
     - `DB_HOST`: `mysql-service` (use the Service name!)
     - `DB_PORT`: `3306`
     - `DB_USER`: `appuser`
     - `DB_PASSWORD`: `apppass`

---

### Steps

1. Create three files: `mysql-pod.yaml`, `mysql-service.yaml`, `webapp-pod.yaml`
2. Write the YAMLs using what you've learned
3. Apply them in order:
   ```bash
   kubectl apply -f mysql-pod.yaml
   kubectl apply -f mysql-service.yaml
   kubectl apply -f webapp-pod.yaml
   ```
4. Verify everything:
   ```bash
   kubectl get pods
   kubectl get svc
   kubectl get endpoints mysql-service
   ```

---

### Success Criteria

✅ **Database Pod is running**  
✅ **Service has endpoints** (meaning it found the Pod)  
✅ **Application Pod is running**  
✅ **You can explain**:
   - What the Pod is responsible for
   - What the Service is responsible for
   - Why we use the Service name instead of Pod IP
   - How the Service finds Pods (using labels/selectors)

---

## Solutions (Try First!)

<details>
<summary>Click to reveal mysql-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
    tier: database
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "rootpass"
    - name: MYSQL_DATABASE
      value: "myapp"
    - name: MYSQL_USER
      value: "appuser"
    - name: MYSQL_PASSWORD
      value: "apppass"
```

</details>

<details>
<summary>Click to reveal mysql-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```

</details>

<details>
<summary>Click to reveal webapp-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    app: webapp
    tier: frontend
spec:
  containers:
  - name: webapp
    image: nginx:1.25
    env:
    - name: DB_HOST
      value: "mysql-service"
    - name: DB_PORT
      value: "3306"
    - name: DB_USER
      value: "appuser"
    - name: DB_PASSWORD
      value: "apppass"
```

</details>

---

## Verification Commands

After completing the exercise:

```bash
# Check all Pods are running
kubectl get pods

# Check Service exists and has a ClusterIP
kubectl get svc mysql-service

# Verify Service found the database Pod
kubectl get endpoints mysql-service

# Check the webapp's environment variables
kubectl exec webapp-pod -- env | grep DB_

# Test DNS resolution from the webapp
kubectl exec webapp-pod -- nslookup mysql-service
```

---

## Cleanup

When you're done:

```bash
# Stage 1 & 2
kubectl delete pod postgres-pod app-pod
kubectl delete svc postgres-service

# Stage 3
kubectl delete pod mysql-pod webapp-pod
kubectl delete svc mysql-service
```

---

## Key Takeaways

1. **Services provide stable endpoints** — Pod IPs change, Service IPs don't
2. **Services use selectors to find Pods** — they match labels automatically
3. **Service names become DNS entries** — use the Service name, not IPs
4. **Services don't run containers** — they're routing rules, not workloads
5. **Clear responsibilities**:
   - **Pod**: Runs the actual application/database
   - **Service**: Provides stable networking to reach the Pod
6. **Port mapping**: `port` (what clients use) → `targetPort` (Pod's container port)
7. **Endpoints show the truth** — they reveal which Pod IPs the Service is routing to

---

## Common Mistakes to Avoid

❌ **Hardcoding Pod IPs** — always use Service names  
❌ **Forgetting labels on Pods** — Services can't find Pods without matching labels  
❌ **Mismatched selectors** — Service selector must match Pod labels exactly  
❌ **Wrong targetPort** — must match the container's `containerPort`  
❌ **Thinking Services run containers** — they don't, they're just routing rules

---

## What's Next?

- **Deployments**: Managing multiple replicas of Pods
- **Service Types**: NodePort and LoadBalancer for external access
- **ConfigMaps and Secrets**: Better ways to manage configuration
- **Namespaces**: Organizing resources in larger clusters

But first, make sure you understand the Pod ↔ Service relationship! 🚀
