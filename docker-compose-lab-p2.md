


# Lab 2 — Startup Order Is NOT Readiness  
**Why `depends_on` Exists — and Why It Is Not Enough**

**CLI-first | Mentor-guided | Explicit reasoning | Windows + Linux/macOS**

> Hi 👋  
> In this lab, you are going to experience a very common real-world failure:
>
> > “Everything is running, but the application still crashes.”
>
> This lab will teach you **why that happens**, **what `depends_on` really does**,  
> and **why startup order is not the same as service readiness**.
>
> Nothing here is theoretical.  
> You will **see the failure**, **prove it**, and **understand it**.

---

## What you will learn (explicit outcomes)

By the end of this lab, you will be able to explain:

1. What `depends_on` **actually guarantees**
2. What `depends_on` **does NOT guarantee**
3. The difference between:
   - *container started*
   - *service ready*
4. Why many Compose setups “work on the second try”
5. How to prove startup problems using logs and timing

---

## Prerequisites

- You completed **Lab 1**
- You understand:
  - services vs containers
  - Compose project lifecycle
  - internal service DNS (service name resolution)
- You are comfortable with:
  - `docker compose up`
  - `docker compose logs`
  - `docker compose exec`

---

## Important rule for this lab

🚫 You are **not allowed** to “fix” the problem early  
🚫 You are **not allowed** to restart manually to make it work  

You must first **observe, prove, and understand the failure**.

---

# Part A — Create a system that fails at startup (on purpose)

> Goal: Build a system where the application depends on the database,  
> but the database is **not ready in time**.

We will use:
- `postgres` as a database (slow startup, realistic)
- `busybox` as a fake “app” that immediately tries to connect

---

## A1) Create a new working folder

### Linux/macOS
```bash
mkdir compose-lab-2 && cd compose-lab-2
````

### Windows PowerShell

```powershell
mkdir compose-lab-2
cd compose-lab-2
```

---

## A2) Create a Compose file WITHOUT `depends_on`

Create a file named `compose.yaml`.

### Linux/macOS (bash/zsh)

```bash
cat > compose.yaml <<'YAML'
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo

  app:
    image: busybox
    command: >
      sh -c "
      echo 'APP: trying to connect to database...';
      nc -z db 5432;
      echo 'APP: database is reachable';
      "
YAML
```

### Windows PowerShell

```powershell
@"
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo

  app:
    image: busybox
    command: >
      sh -c "
      echo 'APP: trying to connect to database...';
      nc -z db 5432;
      echo 'APP: database is reachable';
      "
"@ | Out-File -Encoding utf8 compose.yaml
```

---

## A3) Start the system

### Linux/macOS / Windows PowerShell

```bash
docker compose up
```

---

## A4) Observe what happens (do NOT fix yet)

Look carefully at the output.

You should see something like:

* Postgres starting
* App starting immediately
* App exiting with failure

### Check containers state

```bash
docker compose ps
```

✅ **You should see**

* `db` → running
* `app` → exited (failed)

---

## A5) Inspect logs separately (this is critical)

### App logs

```bash
docker compose logs app
```

You should see:

```
APP: trying to connect to database...
```

…and then nothing else.

### Database logs

```bash
docker compose logs db
```

You should see:

* database initialization
* startup messages
* readiness happening **after** the app already tried

---

## Explicit conclusion (stop and read)

> At this point:
>
> * The database container **was started**
> * The application container **was started**
> * The application **still failed**
>
> This proves:
>
> ❌ “Container started” does NOT mean “service ready”

This is **not a bug**.
This is **exactly how containers work**.

---

# Part B — Add `depends_on` (and see what changes)

> Now we will add `depends_on`.
> Many people expect this to fix everything.
> You are about to see what it *actually* fixes.

---

## B1) Update `compose.yaml` to include `depends_on`

Edit your `compose.yaml` so it looks like this:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo

  app:
    image: busybox
    depends_on:
      - db
    command: >
      sh -c "
      echo 'APP: trying to connect to database...';
      nc -z db 5432;
      echo 'APP: database is reachable';
      "
```

---

## B2) Start again from scratch

⚠️ This is important: we must start clean.

```bash
docker compose down
docker compose up
```

---

## B3) Observe carefully (again)

### Check container status

```bash
docker compose ps
```

✅ **You should still see**

* `db` → running
* `app` → exited

### Check logs again

```bash
docker compose logs app
```

You should see the **same failure**.

---

## Explicit conclusion (very important)

> `depends_on` did exactly what it promised:
>
> ✔ It waited until the **db container started**
>
> ❌ It did NOT wait until the **database was ready to accept connections**

This is the single most important Compose lesson in this lab.

---

# Part C — Understand WHY this is correct behavior

Read this slowly.

> Docker Compose does **not know**:
>
> * how long Postgres needs to initialize
> * what “ready” means for your application
> * how to test readiness safely
>
> So Compose **cannot guess**.

Therefore:

> `depends_on` controls **order**, not **readiness**.

This is intentional design, not a limitation.

---

# Part D — Prove readiness timing with a manual check

> Now you will verify readiness manually, from inside a container.

---

## D1) Start only the database

```bash
docker compose down
docker compose up -d db
```

---

## D2) Exec into the database container

```bash
docker compose exec db sh
```

Inside the container, try:

```sh
pg_isready -U demo
```

You may see:

* “no response” at first
* then “accepting connections”

Exit:

```sh
exit
```

---

## D3) Start the app AFTER readiness

```bash
docker compose up app
```

Now check logs:

```bash
docker compose logs app
```

✅ **You should now see**

```
APP: trying to connect to database...
APP: database is reachable
```

---

## Explicit conclusion (say this out loud)

> The app did not need a different image.
> The app did not need a different network.
> The app needed the **database to be ready**, not just started.

---

# Part E — What you should remember from this lab

If you can explain these points, you understood the lab correctly:

1. `depends_on` guarantees **startup order**, nothing more
2. A running container does not imply a ready service
3. Startup bugs are **timing bugs**
4. Compose behaves deterministically — it does exactly what you asked
5. Readiness must be handled explicitly (next lab)

---


