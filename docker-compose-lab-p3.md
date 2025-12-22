
# Lab 3 — Healthchecks: When Compose *Can* Wait (and When It Still Won’t)
**CLI-first | Mentor-guided | Explicit cause → effect | Windows + Linux/macOS**

> Hi 👋  
> In Lab 2 you learned an uncomfortable truth:
>
> > “`depends_on` does not wait for readiness.”
>
> In this lab, you will learn **the only built-in mechanism Compose has to reason about readiness**:
>
> 👉 **healthchecks**
>
> But just as important:
>
> ❗ You will also learn **their limits**, so you don’t build false confidence.
>
> This lab will make you *precise*, not optimistic.

---

## What you will learn (explicit outcomes)

By the end of this lab, you will be able to explain:

1. What a Docker **healthcheck** really is
2. Who runs the healthcheck (and who does not)
3. How Compose can use health status in `depends_on`
4. Why healthchecks improve startup reliability
5. Why healthchecks are **not orchestration**
6. How to inspect health state using CLI tools

---

## Prerequisites

- You completed **Lab 1** and **Lab 2**
- You understand:
  - service startup vs service readiness
  - why `depends_on` alone is insufficient
- You can use:
  - `docker compose logs`
  - `docker compose exec`
  - `docker inspect`

---

## Important mindset for this lab

> A healthcheck is **a signal**, not a guarantee.
>
> It answers:
> “Does this container *appear* healthy right now?”
>
> It does NOT answer:
> “Will this service stay healthy?”
> “Can it handle real traffic?”
> “Is the system safe to proceed forever?”

Keep this distinction in mind throughout the lab.

---

# Part A — Add a healthcheck to the database

> Goal: Teach Docker how to determine when the database is **ready**, not just running.

We will continue with:
- `postgres` (because it has a native readiness command)

---

## A1) Start from a clean state

```bash
docker compose down
````

Make sure nothing is running:

```bash
docker compose ps
```

---

## A2) Add a healthcheck to the database service

Edit your `compose.yaml` so it looks like this:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U demo']
      interval: 5s
      timeout: 5s
      retries: 5
  app:
    image: busybox
    depends_on:
      db:
        condition: service_healthy
    command: >
      sh -c "
      echo 'APP: waiting for database...';
      nc -z db 5432;
      echo 'APP: database is reachable';
      "
```

---

### What this healthcheck does

* **test**: runs `pg_isready -U demo` to verify PostgreSQL accepts connections
  * `pg_isready`: PostgreSQL utility that checks if the server is accepting connections
  * `-U demo`: specifies the database user to connect as (must match `POSTGRES_USER`)
* **interval**: checks every 5 seconds
  * Docker waits 5 seconds between each healthcheck execution
* **timeout**: each check must complete within 5 seconds
  * If `pg_isready` takes longer than 5 seconds, the check is considered failed
* **retries**: marks unhealthy after 5 consecutive failures
  * The container starts in `starting` state
  * After 5 successful checks, it becomes `healthy`
  * After 5 failed checks, it becomes `unhealthy`
* **result**: Docker reports the container as `healthy` only when the check succeeds
  * This status can be queried via `docker inspect` or `docker compose ps`
  * Compose can use this status in `depends_on` conditions

## A3) Read this carefully before running

You have just expressed **new information** to Compose:

* What “healthy” means for the database
* That the app should wait for that condition

This is **not magic**.
This is **explicit logic you defined**.

---

## A4) Start the system

```bash
docker compose up
```

---

## A5) Observe the behavior (this time it should succeed)

### Check container status

```bash
docker compose ps
```

✅ **You should see**

* `db` → running (healthy)
* `app` → exited (successfully)

---

## A6) Inspect logs (prove what happened)

### Database logs

```bash
docker compose logs db
```

You should see:

* database startup
* readiness messages

### App logs

```bash
docker compose logs app
```

You should see:

```
APP: waiting for database...
APP: database is reachable
```

---

## Explicit conclusion (very important)

> This time, the app waited **not because of timing**,
> but because you gave Compose a **readiness signal**.

This is the **first time** in this course that:

* order
* readiness
* and dependency
  are aligned correctly.

---

# Part B — Inspect health status directly (no assumptions)

> Never trust “it worked”.
> Always verify health state explicitly.

---

## B1) Inspect container health via Docker CLI

First, find the database container name:

```bash
docker compose ps
```

Then inspect it:

```bash
docker inspect $(docker compose ps -q db)
```

Look for this section in the output:

```json
"Health": {
  "Status": "healthy",
  ...
}
```

✅ **You must confirm** that:

* `Status` is `healthy`
* healthchecks ran multiple times

---

## B2) Watch health transitions in real time

Restart everything:

```bash
docker compose down
docker compose up
```

In another terminal, run:

```bash
docker events --filter container=$(docker compose ps -q db)
```

✅ **You should see**

* health_status: starting
* then health_status: healthy

---

## Explicit conclusion

> Healthchecks are **evaluated continuously**, not once.
>
> Docker is constantly asking:
> “Are you still healthy?”

---

# Part C — Understand the limits of healthchecks

> This section is critical.
> Many people stop thinking too early here.

---

## C0) Make the failure observable (temporary change)

Up to now, your `app` service is designed to exit successfully after the first connection check.
That is correct for Parts A and B.

For Part C, we want the app to keep running so you can observe what happens *after* the database fails.

Temporarily edit `compose.yaml` and replace the `app` command with a loop:

```yaml
  app:
    image: busybox
    depends_on:
      db:
        condition: service_healthy
    command: >
      sh -c "
      while true; do
        echo 'APP: checking database reachability...';
        if nc -z db 5432; then
          echo 'APP: database is reachable';
        else
          echo 'APP: database is NOT reachable';
        fi
        sleep 2;
      done
      "
```

If your `db` service has a restart policy (like `restart: always`), temporarily set it to:

```yaml
  db:
    restart: "no"
```

Apply the change without tearing down volumes:

```bash
docker compose up -d --force-recreate app
```

---

## C1) Healthchecks do NOT protect against future failure

Let’s prove it.

### Kill the database process manually

```bash
docker compose exec db sh
```

Inside the container:

```sh
pkill postgres
```

Exit:

```sh
exit
```

---

## C2) Observe what happens

Check container status:

```bash
docker compose ps
```

Watch the app logs:

```bash
docker compose logs -f app
```

Check logs:

```bash
docker compose logs db
```

You may see:

* the `db` container exiting (common when the main Postgres process is killed)
* OR the `db` container still running but health status becoming `unhealthy`
* the `app` container continuing to run and printing `database is NOT reachable`

But:

❗ **Compose will NOT restart dependent services automatically**

---

## Explicit conclusion (very important)

> Healthchecks inform Docker.
>
> They do NOT:
>
> * restart other services
> * re-run dependency logic
> * heal the system automatically
>
> Compose is **not an orchestrator**.

---

# Part D — Why this is still correct design

Read this carefully.

> Docker Compose is a **local development and system definition tool**.
>
> It is not responsible for:
>
> * high availability
> * automatic failover
> * continuous reconciliation
>
> That responsibility belongs to:
>
> * Kubernetes
> * Nomad
> * Swarm (historically)

Compose is intentionally simpler.

---

# Part E — What you should remember from this lab

You fully understood this lab if you can explain:

1. A healthcheck is a **command run by Docker**, not by Compose
2. Health status is **observational**, not corrective
3. `depends_on: condition: service_healthy` waits only at startup
4. Healthchecks improve reliability, but do not guarantee stability
5. Compose remains deterministic and explicit

---


