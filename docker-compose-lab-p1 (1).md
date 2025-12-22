
# Lab 1 — From “I can manage with `docker run`” to “Compose is the system”
**Beginner-friendly | CLI-first | Mentor-guided | Windows + Linux/macOS**

> Hi 👋  
> In this lab you will **prove with your own hands** why Docker Compose is not “just syntax”.  
> You already know Dockerfile, networks, and volumes — great. Now we’ll build the missing mental model:
>
> ✅ **Dockerfile builds ONE image**  
> ✅ **`docker run` starts ONE container**  
> ✅ **Docker Compose defines ONE SYSTEM** (multiple services that must live together)

---

## What you will learn (explicit outcomes)
By the end of this lab you will be able to say, with evidence:

1. Why `--name` is not the same as a **service**
2. Why a “working setup” in CLI can still be **fragile / non-reproducible**
3. How Compose creates a **project** (a coherent system) and manages lifecycle as a unit
4. How to run the **same system twice** on one machine (without mental chaos)
5. How to verify everything using **inspection tools** (not guesses)

---

## Prerequisites
- Docker Desktop installed (Windows/macOS) or Docker Engine (Linux)
- You can run `docker ps` and see output
- You already understand:
  - containers do not communicate unless networked
  - ports: internal vs published
  - volume persistence (we won’t focus on volumes yet)

---

## Rules of this lab
- You will do everything via terminal.
- **Do not skip the “Check” steps.** The checks are the learning.
- When I say “you should see…”, I mean **verify it**.

---

## Folder setup (same for everyone)

### Linux/macOS (bash/zsh)
```bash
mkdir -p compose-lab-1 && cd compose-lab-1
````

### Windows PowerShell

```powershell
mkdir compose-lab-1
cd compose-lab-1
```

---

# Part A — Build a “working” two-container system with pure CLI

> Goal: You will create a system that works… and then discover why it’s still not “a system” you can trust and repeat.

We will use:

* **redis** as a database-like service (simple, fast, stable)
* **redis-commander** as an admin UI (web UI that must talk to redis)

This is perfect for Compose learning because:

* there is a clear dependency (UI needs DB)
* networking matters (service-to-service communication)
* it’s easy to inspect

---

## A1) Create a dedicated Docker network (manual infrastructure)

> You already know networking — now you’ll use it to build a “stack” manually.

### Linux/macOS (bash/zsh)

```bash
docker network create labnet
docker network ls | grep labnet
# Explanation:
# - `docker network ls`: Lists all Docker networks on your system
# - `|`: Pipe operator - sends output from left command to right command
# - `grep labnet`: Filters the list to show only lines containing "labnet"
# This command verifies that the "labnet" network was successfully created
```

### Windows PowerShell

```powershell
docker network create labnet
docker network ls | Select-String labnet
# Explanation:
# - `docker network ls`: Lists all Docker networks on your system
# - `|`: Pipe operator - sends output from left command to right command
# - `Select-String labnet`: PowerShell cmdlet that filters the list to show only lines containing "labnet"
# This command verifies that the "labnet" network was successfully created
```

✅ **You should see** a network named `labnet`.

---

## A2) Start Redis (db) manually

### Linux/macOS (bash/zsh)

```bash
docker run -d --name lab-db --network labnet redis:7-alpine
docker ps --filter name=lab-db
# Explanation:
# - `docker ps`: Lists all running Docker containers
# - `--filter name=lab-db`: Filters the list to show only containers whose name contains "lab-db"
# This command verifies that the "lab-db" container is running
```

### Windows PowerShell

```powershell
docker run -d --name lab-db --network labnet redis:7-alpine
docker ps --filter name=lab-db
```

✅ **You should see** container `lab-db` running.

### Check (what did Docker attach to the network?)

> You are not guessing. You are proving.

#### Linux/macOS

```bash
docker network inspect labnet | grep -A3 '"Name": "lab-db"'
# Explanation:
# - `docker network inspect labnet`: Shows detailed information about the "labnet" network
# - `grep -A3 '"Name": "lab-db"'`: Finds the line with "Name": "lab-db" and shows 3 lines after it
# This shows the container's network configuration
```

#### Windows PowerShell (simpler check)

```powershell
docker network inspect labnet | Select-String "lab-db"
```

✅ **You should see** evidence that `lab-db` is connected to `labnet`.

---

## A3) Start Redis Commander (web UI) manually

This UI must connect to Redis using the hostname `lab-db` (container name) over the Docker network.

### Linux/macOS (bash/zsh)

```bash
docker run -d --name lab-ui --network labnet \
  -e REDIS_HOSTS=local:lab-db:6379 \
  -p 8081:8081 \
  rediscommander/redis-commander:latest
# Explanation:
# - `docker run -d`: Runs a container in detached mode (background)
# - `--name lab-ui`: Names the container "lab-ui" for easy reference
# - `--network labnet`: Connects the container to the "labnet" network (same network as lab-db)
# - `-e REDIS_HOSTS=local:lab-db:6379`: Sets environment variable telling the UI where to find Redis
#   - "local" is an alias, "lab-db" is the container hostname, "6379" is Redis default port
# - `-p 8081:8081`: Maps host port 8081 to container port 8081 (access UI at localhost:8081)
# - `rediscommander/redis-commander:latest`: The Docker image to run

docker ps --filter name=lab-ui
```

### Windows PowerShell

```powershell
docker run -d --name lab-ui --network labnet `
  -e REDIS_HOSTS=local:lab-db:6379 `
  -p 8081:8081 `
  rediscommander/redis-commander:latest

docker ps --filter name=lab-ui
```

✅ **You should see** `lab-ui` running and port mapping `8081->8081`.

### Check 1: logs (did it connect?)

#### Linux/macOS

```bash
docker logs --tail 30 lab-ui
```

#### Windows PowerShell

```powershell
docker logs --tail 30 lab-ui
```

✅ **You should see** logs that indicate the UI is running (and typically it will connect to Redis without errors).

### Check 2: test from inside the UI container

> This is the most important kind of check: **inside the container, by hostname**.

#### Linux/macOS

```bash
docker exec -it lab-ui sh
```

#### Windows PowerShell

```powershell
docker exec -it lab-ui sh
```

Now inside the container, run:

```sh
ping -c 1 lab-db
```

✅ **You should see** the ping succeed.

Exit the container:

```sh
exit
```

### Check 3: host access (published port)

Open in browser:

* `http://localhost:8081`

✅ **You should see** Redis Commander UI page.

---

## A4) Stop and start the system (manual lifecycle)

> Here is your first pain: you have to remember what belongs to what.

### Stop both containers

#### Linux/macOS

```bash
docker stop lab-ui lab-db
docker ps --filter name=lab-
```

#### Windows PowerShell

```powershell
docker stop lab-ui lab-db
docker ps --filter name=lab-
```

✅ **You should see** no running containers matching `lab-` (or they are “Exited”).

### Start both containers again

#### Linux/macOS

```bash
docker start lab-db lab-ui
docker ps --filter name=lab-
```

#### Windows PowerShell

```powershell
docker start lab-db lab-ui
docker ps --filter name=lab-
```

✅ **You should see** both running again.

---

# Part B — Now I will force the “Compose threshold”

> Now you will experience something important:
>
> **A manual CLI setup can “work”, yet still be a bad system design.**
>
> You will *try to run the same system twice* on the same machine.

---

## B1) Try to run a second copy (naive approach)

You will attempt:

* a second redis
* a second UI
* same network logic

### Try to run second redis with same name (should fail)

#### Linux/macOS

```bash
docker run -d --name lab-db --network labnet redis:7-alpine
```

#### Windows PowerShell

```powershell
docker run -d --name lab-db --network labnet redis:7-alpine
```

✅ **You should see** an error like:

* conflict: container name "lab-db" is already in use

### Reflection (important)

You might think:

> “Fine, I’ll just use a different name.”

That’s exactly the trap.

Yes, you can rename containers.
But now you must keep a mental map of:

* which name belongs to which system
* which UI connects to which DB
* which ports belong to which copy
* what to stop / remove later

This is the point: **you’re managing instances, not a system definition.**

---

## B2) Try the “rename everything” workaround (and feel the pain)

Let’s do it anyway so you *feel* it.

### Start second DB and UI with new names and new host port

#### Linux/macOS (bash/zsh)

```bash
docker run -d --name lab2-db --network labnet redis:7-alpine

docker run -d --name lab2-ui --network labnet \
  -e REDIS_HOSTS=local:lab2-db:6379 \
  -p 8082:8081 \
  rediscommander/redis-commander:latest
```

#### Windows PowerShell

```powershell
docker run -d --name lab2-db --network labnet redis:7-alpine

docker run -d --name lab2-ui --network labnet `
  -e REDIS_HOSTS=local:lab2-db:6379 `
  -p 8082:8081 `
  rediscommander/redis-commander:latest
```

✅ **You should see**

* `lab2-ui` publishes `8082->8081`
* both copies running in `docker ps`

### Check (prove you now have two independent copies)

* Open:

  * `http://localhost:8081` (copy 1 UI)
  * `http://localhost:8082` (copy 2 UI)

✅ **You should see** both UIs accessible.

---

## B3) The “system confusion” checkpoint (this is the lesson)

Now answer honestly:

* If I say “bring down the whole system #1”, what exactly do you type?
* If I say “restart only the UI in system #2”, what exactly do you type?
* If I say “remove everything cleanly, no leftovers”, are you sure you won’t miss something?

You *can* do it manually.
But it becomes fragile because the “system” exists only in **your memory**.

**This is the exact problem Compose solves.**

---

# Part C — Translate the SAME system into Docker Compose (the correct abstraction)

> Now you will create a Compose file that represents **the system**, not the containers.

## C1) Create `compose.yaml`

Create a file named `compose.yaml` in your folder.

### Linux/macOS (bash/zsh) quick create

```bash
cat > compose.yaml <<'YAML'
services:
  db:
    image: redis:7-alpine

  ui:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:db:6379
    ports:
      - "8081:8081"
YAML
```

### Windows PowerShell quick create

```powershell
@"
services:
  db:
    image: redis:7-alpine

  ui:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:db:6379
    ports:
      - "8081:8081"
"@ | Out-File -Encoding utf8 compose.yaml
```

✅ **Important observation you must understand:**

* We did **NOT** define networks.
* We did **NOT** define container names.
* We connected UI→DB using `db` **service name**.

---

## C2) Start the system with Compose

### Linux/macOS / Windows PowerShell

```bash
docker compose up -d
docker compose ps
```

✅ **You should see**

* two services: `db` and `ui`
* both running

---

## C3) Inspect what Compose created (prove it)

### 1) Compose created a network for you

#### Linux/macOS

```bash
docker network ls | grep compose-lab-1
```

#### Windows PowerShell

```powershell
docker network ls | Select-String compose-lab-1
```

✅ **You should see** a network whose name includes your folder/project name (example: `compose-lab-1_default`).

### 2) Containers got names based on PROJECT + SERVICE

#### Linux/macOS / Windows PowerShell

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
```

✅ **You should see** container names like:

* `compose-lab-1-db-1`
* `compose-lab-1-ui-1`

> **This is the conclusion:**
> You didn’t *choose* runtime container names.
> You declared **roles (services)** and Compose created the instances predictably.

---

## C4) Lifecycle as a system (this is the big win)

Now do these commands and notice how your brain feels:

### View logs for a service

```bash
docker compose logs --tail 30 ui
```

### Exec into a service container (without memorizing container IDs)

```bash
docker compose exec ui sh
```

Inside the UI container:

```sh
ping -c 1 db
exit
```

✅ **You should see**

* `db` resolves automatically
* you didn’t need `--network` or manual DNS thinking

### Bring the whole system down cleanly

```bash
docker compose down
docker compose ps
```

✅ **You should see**

* nothing running for this project

> **Explicit conclusion:**
> With Compose, “the system” is not in your memory.
> It is written in `compose.yaml`, and the CLI operates on it as a unit.

---

# Part D — The “run it twice” proof (the Compose killer feature)

> Now you will run the SAME system twice with **two project names**.
> This is where `--name` hacks completely lose to system-level design.

## D1) Start system #1 with explicit project name

### Linux/macOS / Windows PowerShell

```bash
docker compose -p stack1 up -d
# Explanation:
# - `-p stack1`: Sets the project name to "stack1" (overrides folder-based default)
#   - All resources (containers, networks, volumes) get prefixed with "stack1"
#   - Enables running multiple isolated copies of the same compose.yaml
# - `up -d`: Starts all services in detached mode (background)

docker compose -p stack1 ps
# Explanation:
# - `-p stack1`: Targets the "stack1" project specifically
# - `ps`: Shows status of containers belonging to this project
```

## D2) Start system #2 with a different project name and different host port

We need to avoid port conflict on the host.
So we’ll override only the published port for stack2.

### Linux/macOS (bash/zsh)

```bash
#### Attempt to start stack2 (this will fail - intentionally)

```bash
docker compose -p stack2 up -d
```

❌ **You should see** an error about port 8081 already being in use.

> **This is the teaching moment:** Two projects cannot share the same host port.
> You must override the port mapping for stack2.

#### Fix: Start stack2 with port override

Now we need to handle port mapping correctly for stack2.

**Important concept:** When you use multiple Compose files (`-f compose.yaml -f compose.stack2.yaml`), Docker Compose **merges** the service definitions. To override the port, we need to completely redefine the `ports` section in the override file.

We'll show two valid solutions:

---

### Solution 1: Environment-agnostic base file (Best Practice)

Remove the `ports` section from `compose.yaml` and define host ports only in stack-specific files.

#### Update base file

##### Linux/macOS (bash/zsh)

```bash
cat > compose.yaml <<'YAML'
services:
  db:
    image: redis:7-alpine

  ui:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:db:6379
YAML
```

##### Windows PowerShell

```powershell
@"
services:
  db:
    image: redis:7-alpine

  ui:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:db:6379
"@ | Out-File -Encoding utf8 compose.yaml
```

#### Create stack1 override

##### Linux/macOS (bash/zsh)

```bash
cat > compose.stack1.yaml <<'YAML'
services:
  ui:
    ports:
      - "8081:8081"
YAML
```

##### Windows PowerShell

```powershell
@"
services:
  ui:
    ports:
      - "8081:8081"
"@ | Out-File -Encoding utf8 compose.stack1.yaml
```

#### Create stack2 override

##### Linux/macOS (bash/zsh)

```bash
cat > compose.stack2.yaml <<'YAML'
services:
  ui:
    ports:
      - "8082:8081"
YAML
```

##### Windows PowerShell

```powershell
@"
services:
  ui:
    ports:
      - "8082:8081"
"@ | Out-File -Encoding utf8 compose.stack2.yaml
```

#### Start both stacks

```bash
docker compose -p stack1 -f compose.yaml -f compose.stack1.yaml up -d
docker compose -p stack2 -f compose.yaml -f compose.stack2.yaml up -d

docker compose -p stack1 ps
docker compose -p stack2 ps
```

---

### Solution 2: Parameterized base file (Alternative)

Keep ports in base file but use environment variables for flexibility.

#### Update base file with variable

##### Linux/macOS (bash/zsh)

```bash
cat > compose.yaml <<'YAML'
services:
  db:
    image: redis:7-alpine

  ui:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:db:6379
    ports:
      - "${UI_PORT:-8081}:8081"
YAML
```

##### Windows PowerShell

```powershell
@"
services:
  db:
    image: redis:7-alpine

  ui:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:db:6379
    ports:
      - "`${UI_PORT:-8081}:8081"
"@ | Out-File -Encoding utf8 compose.yaml
```

#### Start both stacks with environment variables

##### Linux/macOS (bash/zsh)

```bash
UI_PORT=8081 docker compose -p stack1 up -d
UI_PORT=8082 docker compose -p stack2 up -d

docker compose -p stack1 ps
docker compose -p stack2 ps
```

##### Windows PowerShell

```powershell
$env:UI_PORT=8081; docker compose -p stack1 up -d
$env:UI_PORT=8082; docker compose -p stack2 up -d

docker compose -p stack1 ps
docker compose -p stack2 ps
```

---

✅ **You should now have**

* stack1 UI on: `http://localhost:8081`
* stack2 UI on: `http://localhost:8082`

### Check container names (they won’t collide)

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

✅ **You should see** names like:

* `stack1-db-1`, `stack1-ui-1`
* `stack2-db-1`, `stack2-ui-1`

> **Explicit conclusion (this is the lesson you must remember):**
> With Compose, running the same system twice is a **first-class feature**.
> Without Compose, you can do it, but you are forced to manage everything manually:
> names, ports, networks, and cleanup rules in your head.

---

# Cleanup (do this carefully)

### Stop and remove both projects

```bash
docker compose -p stack1 down
docker compose -p stack2 -f compose.yaml -f compose.stack2.yaml down
```

### Remove the manual containers + manual network (from Part A/B), if still exist

#### Linux/macOS / Windows PowerShell

```bash
docker rm -f lab-ui lab-db lab2-ui lab2-db 2>/dev/null || true
docker network rm labnet 2>/dev/null || true
```

Windows PowerShell alternative if you see errors:

```powershell
docker rm -f lab-ui lab-db lab2-ui lab2-db 2>$null
docker network rm labnet 2>$null
```

✅ Final check:

```bash
docker ps
docker network ls
```

---

# What you learned (state it explicitly to yourself)

If you can explain these 5 points, you understood the lab:

1. **A container name is a runtime detail.** It does not define a system.
2. **A service name is a role** inside a declared system.
3. Compose creates a **project boundary**: containers + network + lifecycle tools.
4. Compose lets you manage the system as one unit: `up`, `down`, `logs`, `exec`.
5. Compose makes “run it twice” clean and predictable via `-p` (project name).

---


