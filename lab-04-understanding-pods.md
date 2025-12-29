# Lab 04: Understanding Pods

**Goal**: Learn what a Pod is, understand its core structure, and write Pod YAML files independently.

**Prerequisites**: Basic Docker knowledge (images, containers, environment variables)

**Duration**: 30-45 minutes

---

## Before You Begin: Reading Kubernetes YAML

Before we create our first Pod, let's learn how to **read** Kubernetes YAML files. Understanding YAML structure will help you make sense of the Pod definitions you're about to see.

### The Core Mental Model

**YAML indentation represents hierarchy and ownership.**

Every time you see indentation in a YAML file, it's answering the question: **"What belongs to what?"**

Think of it like a family tree:
- Top-level items are independent
- Indented items belong to their parent
- Deeper indentation means nested responsibility

---

### Reading Structure: What Belongs to What?

Here's a preview of the Pod YAML you'll see in a moment:

```yaml
apiVersion: v1           # Top-level: independent field
kind: Pod                # Top-level: independent field
metadata:                # Top-level: independent field
  name: nginx-pod        #   ↳ belongs to metadata
spec:                    # Top-level: independent field
  containers:            #   ↳ belongs to spec
  - name: nginx-container#     ↳ belongs to containers (list item)
    image: nginx:1.25    #       ↳ belongs to this container
```

**Key Insight**: The indentation tells you the **responsibility chain**.

---

### The Four Pillars of a Pod

Every Pod YAML has four top-level sections (no indentation):

```yaml
apiVersion: v1
kind: Pod
metadata:
spec:
```

These are all at the same level because they're all direct properties of the Pod resource.

**What each section owns**:
- `metadata` owns identifying information (like `name`)
- `spec` owns the specification (like `containers`)

---

### Understanding Lists vs Objects

You'll see two types of structures in Kubernetes YAML:

#### Object (Key-Value Pairs)

```yaml
metadata:
  name: nginx-pod
```

- `metadata` is an **object** (it contains key-value pairs)
- `name` is a **property** of that object
- No `-` (dash) needed

#### List (Array of Items)

```yaml
containers:
- name: nginx-container
  image: nginx:1.25
```

- `containers` is a **list** (it can contain multiple containers)
- The `-` (dash) starts a new list item
- `name` and `image` belong to that specific container

**Mental Model**:
- **Object**: "This thing has properties"
- **List**: "This is a collection of things"

**Why it matters**: The `-` tells you "this is one item in a collection." Everything indented under the `-` belongs to that item.

---

### Ownership Chain Example

Let's look at a slightly more complex example you'll work with later:

```yaml
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
    ports:
    - containerPort: 80
    env:
    - name: ENVIRONMENT
      value: "development"
```

**Reading the ownership**:
1. `spec` owns `containers`
2. `containers` is a list of containers
3. Each container (started by `-`) owns:
   - `name`
   - `image`
   - `ports` (which is itself a list)
   - `env` (which is itself a list)
4. Each port (started by `-`) owns `containerPort`
5. Each env variable (started by `-`) owns `name` and `value`

**Why this structure?**
- Ports belong to a specific container (not to the Pod)
- Environment variables belong to a specific container (not to the Pod)
- If you had multiple containers, each would have its own `ports` and `env`

---

### Visualizing the Hierarchy

```
Pod
├── apiVersion
├── kind
├── metadata
│   └── name
└── spec
    └── containers (list)
        └── container #1
            ├── name
            ├── image
            ├── ports (list)
            │   └── port #1
            │       └── containerPort
            └── env (list)
                ├── env var #1
                │   ├── name
                │   └── value
                └── env var #2
                    ├── name
                    └── value
```

**Each level of indentation = one level deeper in the hierarchy.**

---

### Common Reading Mistakes to Avoid

#### ❌ Mistake 1: Wrong Indentation Level

```yaml
spec:
  containers:
  - name: nginx-container
  image: nginx:1.25  # WRONG: not indented under the container
```

**What this means**: `image` is at the same level as the container item, so it's not a property of the container.

**Correct reading**:
```yaml
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25  # CORRECT: belongs to the container
```

---

#### ❌ Mistake 2: Fields Under the Wrong Parent

```yaml
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
  ports:  # WRONG: ports is at the containers level
  - containerPort: 80
```

**What this means**: `ports` belongs to `spec.containers`, not to the specific container.

**Correct reading**:
```yaml
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
    ports:  # CORRECT: belongs to this container
    - containerPort: 80
```

---

#### ❌ Mistake 3: Missing the `-` for List Items

```yaml
containers:
  name: nginx-container  # WRONG: missing the dash
  image: nginx:1.25
```

**What this means**: Without the `-`, this looks like `containers` is an object with properties, not a list.

**Correct reading**:
```yaml
containers:
- name: nginx-container  # CORRECT: this is a list item
  image: nginx:1.25
```

---

#### ❌ Mistake 4: Inconsistent Indentation

```yaml
spec:
  containers:
  - name: nginx-container
      image: nginx:1.25  # WRONG: too much indentation
```

**What this means**: `image` is indented more than `name`, suggesting it belongs to `name` (which doesn't make sense).

**Correct reading**:
```yaml
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25  # CORRECT: same level as name
```

---

### Quick Reference: Reading YAML Structure

1. **Top-level fields** (apiVersion, kind, metadata, spec): No indentation
2. **Fields that belong to a parent**: Indented 2 spaces
3. **List items**: Look for `-` at the parent's indentation level
4. **Properties of a list item**: Indented under the `-`
5. **Nested lists**: Each list gets its own `-` at the appropriate level

---

### Practice: Reading This Structure

Before moving on, look at this YAML and identify the ownership:

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: "localhost"
    - name: DB_PORT
      value: "5432"
```

**Questions to ask yourself**:
- What owns `containers`? → `spec`
- What owns `env`? → The container named `app`
- How many environment variables are there? → 2 (two `-` under `env`)
- What owns `value`? → Each specific environment variable

---

Now that you understand how to read YAML structure, let's create your first Pod!

---

## Stage 1: Guided — Your First Pod

### What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers and represents a single instance of a running process in your cluster.

Think of it like this:
- **Docker**: You run a container
- **Kubernetes**: You run a Pod (which contains containers)

### Creating Your First Pod

Create a file named `simple-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
```

### Understanding Each Section

Let's break down what each section does:

#### 1. `apiVersion: v1`
- **What it does**: Tells Kubernetes which API version to use
- **Why it's needed**: Different Kubernetes resources use different API versions. Pods use `v1` (the core, stable API)
- **Think of it as**: The "language version" Kubernetes should use to read this file

#### 2. `kind: Pod`
- **What it does**: Declares what type of Kubernetes resource you're creating
- **Why it's needed**: Kubernetes supports many resource types (Pods, Services, Deployments, etc.). This tells it "I want to create a Pod"
- **Think of it as**: The "type" of object you're defining

#### 3. `metadata`
- **What it does**: Provides identifying information about your Pod
- **Why it's needed**: Kubernetes needs to track and reference your Pod by name
- **Key field**: `name` — this is how you'll refer to this Pod in commands
- **Think of it as**: The Pod's identity card

#### 4. `spec`
- **What it does**: Describes the **desired state** — what should run inside this Pod
- **Why it's needed**: This is where you define what containers to run
- **Think of it as**: The blueprint of what your Pod should look like

#### 5. `spec.containers`
- **What it does**: Lists all containers that should run in this Pod
- **Why it's needed**: A Pod must have at least one container
- **Key fields**:
  - `name`: Identifier for this container within the Pod
  - `image`: Which Docker image to run
- **Think of it as**: The actual workload definition

---

### Apply and Observe

**Step 1**: Create the Pod

```bash
kubectl apply -f simple-pod.yaml
```

**Step 2**: Check if it's running

```bash
kubectl get pod nginx-pod
```

You should see:
```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

**Step 3**: Get detailed information

```bash
kubectl describe pod nginx-pod
```

### What to Notice in `describe` Output

Look at the output carefully. You'll see:

**What YOU wrote in the YAML**:
- Name: `nginx-pod`
- Container name: `nginx-container`
- Image: `nginx:1.25`

**What KUBERNETES added automatically**:
- Namespace: `default` (you didn't specify one)
- Labels: (none, because you didn't add any)
- Node: The actual machine where the Pod is running
- IP Address: Internal cluster IP assigned to the Pod
- Status: Current state (Running, Pending, etc.)
- Events: History of what happened (image pulled, container started, etc.)

**Key Insight**: You provide the **desired state** in YAML. Kubernetes handles the **actual state** — scheduling, networking, monitoring.

---

## Stage 2: Guided+ — Extending the Pod

Now let's add more configuration to make our Pod more realistic.

### Adding Ports and Environment Variables

Update `simple-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
    ports:
    - containerPort: 80
      name: http
    env:
    - name: ENVIRONMENT
      value: "development"
    - name: LOG_LEVEL
      value: "info"
```

### Understanding the New Sections

#### `ports`
- **What it does**: Documents which ports the container listens on
- **Why it's needed**: It's **documentation** for Kubernetes and other developers. It doesn't actually open or expose the port (that's what Services do)
- **Where it belongs**: Under `containers` — because ports are a property of the container
- **Key fields**:
  - `containerPort`: The port number the container listens on
  - `name`: Optional label for this port (useful when you have multiple ports)

**Important**: This does NOT make your Pod accessible from outside. It just declares "this container listens on port 80".

#### `env`
- **What it does**: Sets environment variables inside the container
- **Why it's needed**: Containers often need configuration (like database URLs, feature flags, etc.)
- **Where it belongs**: Under `containers` — because environment variables are injected into the container
- **Key fields**:
  - `name`: The environment variable name
  - `value`: The value to set

---

### Apply the Changes

**Step 1**: Delete the old Pod

```bash
kubectl delete pod nginx-pod
```

**Step 2**: Apply the updated YAML

```bash
kubectl apply -f simple-pod.yaml
```

**Step 3**: Describe the Pod again

```bash
kubectl describe pod nginx-pod
```

### What to Notice Now

In the `describe` output, look for:

**Ports section**:
```
Ports:          80/TCP
```

**Environment section**:
```
Environment:
  ENVIRONMENT:  development
  LOG_LEVEL:    info
```

**Key Insight**: These fields are **configuration**, not behavior. They describe what the container needs, but don't create networking rules or external access.

---

## Stage 3: Independent Exercise

Now it's your turn to write a Pod YAML from scratch.

### Task

Create a Pod that runs a PostgreSQL database with the following requirements:

**Requirements**:
1. **Pod name**: `postgres-pod`
2. **Container name**: `postgres`
3. **Image**: `postgres:15`
4. **Port**: PostgreSQL listens on port `5432`
5. **Environment variables** (required by PostgreSQL):
   - `POSTGRES_USER` = `student`
   - `POSTGRES_PASSWORD` = `mypassword`
   - `POSTGRES_DB` = `testdb`

### Steps

1. Create a new file: `postgres-pod.yaml`
2. Write the YAML using the structure you learned
3. Apply it: `kubectl apply -f postgres-pod.yaml`
4. Verify it's running: `kubectl get pod postgres-pod`
5. Check the details: `kubectl describe pod postgres-pod`

### Success Criteria

✅ Your Pod is in `Running` status  
✅ You can explain what each main section does:
   - `apiVersion`
   - `kind`
   - `metadata.name`
   - `spec.containers`
   - `ports`
   - `env`

✅ You understand the difference between what you wrote and what Kubernetes added

---

## Solution (Try First!)

<details>
<summary>Click to reveal solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
spec:
  containers:
  - name: postgres
    image: postgres:15
    ports:
    - containerPort: 5432
      name: postgres
    env:
    - name: POSTGRES_USER
      value: "student"
    - name: POSTGRES_PASSWORD
      value: "mypassword"
    - name: POSTGRES_DB
      value: "testdb"
```

</details>

---

## Verification Commands

After completing the exercise, run these commands:

```bash
# Check Pod status
kubectl get pod postgres-pod

# Get detailed information
kubectl describe pod postgres-pod

# Check logs (you should see PostgreSQL startup messages)
kubectl logs postgres-pod

# Verify environment variables are set
kubectl exec postgres-pod -- env | grep POSTGRES
```

---

## Cleanup

When you're done, delete the Pods:

```bash
kubectl delete pod nginx-pod
kubectl delete pod postgres-pod
```

---

## Key Takeaways

1. **A Pod is the smallest unit** in Kubernetes — it wraps containers
2. **Four core sections** are essential:
   - `apiVersion` — which API to use
   - `kind` — what resource type
   - `metadata` — identity (name, labels)
   - `spec` — desired state (containers)
3. **Containers have configuration**: ports, env, image, name
4. **Ports are documentation** — they don't expose the Pod externally
5. **Kubernetes adds a lot automatically** — you provide the blueprint, it handles the rest
6. **YAML structure matters** — `ports` and `env` belong under `containers` because they configure the container

---

## What's Next?

- **Services**: How to actually expose Pods to network traffic
- **Labels and Selectors**: How to organize and reference Pods
- **Deployments**: How to manage multiple Pod replicas
- **Health Checks**: How to tell Kubernetes if your container is healthy

But first, make sure you can write a Pod YAML confidently! 🚀
