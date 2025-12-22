
# Lab 6 — Debugging Docker Compose Systems Like an Engineer  
**How to Think, Inspect, Prove, and Fix (Not Guess)**

**CLI-first | Mentor-guided | Failure-driven | Windows + Linux/macOS**

> Hi 👋  
> Up to now, everything mostly “worked” once it was designed correctly.
>
> In the real world, things break.
>
> This lab teaches the **most important Compose skill of all**:
>
> 👉 **How to debug a multi-service system methodically**
>
> Not by changing things randomly.  
> Not by restarting until it works.  
> But by **observing, inspecting, and proving**.

---

## What you will learn (explicit outcomes)

By the end of this lab, you will be able to:

1. Follow a **debugging checklist** instead of guessing
2. Use the correct CLI tool for each type of problem
3. Distinguish between:
   - configuration errors
   - networking errors
   - startup/readiness errors
   - data errors
4. Prove *why* something is broken before fixing it
5. Debug Compose systems without touching the YAML first

---

## Prerequisites

- You completed **Labs 1–5**
- You understand:
  - services vs containers
  - networks, ports, volumes
  - depends_on and healthchecks
  - environment variables and `.env`
- You can run:
  - `docker compose ps`
  - `docker compose logs`
  - `docker compose exec`
  - `docker compose config`

---

## The Golden Rule of This Lab

Read this carefully:

> **If you edit `compose.yaml` before you understand the failure,  
> you are not debugging — you are gambling.**

In this lab:
- Failures are **intentional**
- Fixes come **only after proof**

---

# The Debugging Checklist (Memorize This)

You will apply this checklist **every time** something breaks:

1. `docker compose ps`  
   → What is running? What exited?
2. `docker compose logs <service>`  
   → What does the service *say*?
3. `docker compose exec <service>`  
   → What does the service *see*?
4. `docker compose config`  
   → What configuration did Compose actually use?
5. `docker inspect` / `docker network inspect`  
   → What exists at runtime?

You will now practice this **step by step**.

---

# Part A — A Broken System (Wrong Hostname)

> Goal: Debug a networking failure caused by incorrect service naming.

---

## A1) Create a new lab folder

### Linux/macOS
```bash
mkdir compose-lab-6 && cd compose-lab-6
````

### Windows PowerShell

```powershell
mkdir compose-lab-6
cd compose-lab-6
```

---

## A2) Create a deliberately broken Compose file

Create `compose.yaml`:

### Linux/macOS

```bash
cat > compose.yaml <<'YAML'
services:
  db:
    image: redis:7-alpine

  app:
    image: busybox
    command: >
      sh -c "
      echo 'Trying to reach database...';
      nc -z redis 6379;
      echo 'Connected to database';
      "
YAML
```

### Windows PowerShell

```powershell
@"
services:
  db:
    image: redis:7-alpine

  app:
    image: busybox
    command: >
      sh -c "
      echo 'Trying to reach database...';
      nc -z redis 6379;
      echo 'Connected to database';
      "
"@ | Out-File -Encoding utf8 compose.yaml
```

⚠️ **Important:**
The database service is named `db`, but the app tries to reach `redis`.

This is intentional.

---

## A3) Start the system

```bash
docker compose up
```

---

## A4) Step 1 — Inspect container states

```bash
docker compose ps
```

✅ **You should see**

* `db` → running
* `app` → exited

---

## A5) Step 2 — Inspect logs (what does the app say?)

```bash
docker compose logs app
```

You should see:

```
Trying to reach database...
```

…and then nothing.

---

## A6) Step 3 — Exec into the app container

```bash
docker compose exec app sh
```

Inside the container:

```sh
ping -c 1 redis
```

❌ This should fail.

Now try:

```sh
ping -c 1 db
```

✅ This should succeed.

Exit:

```sh
exit
```

---

## Explicit conclusion

> The network is working.
>
> DNS is working.
>
> The failure is **not** Docker networking.
>
> The failure is **wrong service hostname**.

This is a **configuration error**, not an infrastructure problem.

---

## A7) Step 4 — Inspect rendered config

```bash
docker compose config
```

Look carefully:

* Is there any service named `redis`?

❌ No.

---

## Fix (only after proof)

Edit `compose.yaml`:

```yaml
nc -z db 6379;
```

Restart:

```bash
docker compose down
docker compose up
```

✅ The app should now succeed.

---

# Part B — Broken Port Assumption (Internal vs External)

> Goal: Debug confusion between container ports and host ports.

---

## B1) Modify Compose file

Replace `compose.yaml` with:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  client:
    image: busybox
    command: >
      sh -c "
      echo 'Trying localhost:8080';
      nc -z localhost 8080;
      echo 'Connected';
      "
```

---

## B2) Start the system

```bash
docker compose up
```

---

## B3) Inspect container states

```bash
docker compose ps
```

You should see:

* `web` → running
* `client` → exited

---

## B4) Inspect logs

```bash
docker compose logs client
```

❌ The connection fails.

---

## B5) Exec into client container

```bash
docker compose exec client sh
```

Inside:

```sh
nc -z localhost 8080
```

❌ Fails.

Now try:

```sh
nc -z web 80
```

✅ Succeeds.

Exit:

```sh
exit
```

---

## Explicit conclusion

> `ports` expose services to the **host**.
>
> Containers talk to each other using:
>
> * service name
> * internal port
>
> `localhost` inside a container means **the container itself**.

This is a **mental model error**, not a Compose bug.

---

# Part C — Broken Environment Configuration

> Goal: Debug incorrect or missing environment variables.

---

## C1) Replace Compose file again

```yaml
services:
  app:
    image: busybox
    environment:
      APP_MODE: production
    command: >
      sh -c "
      echo 'Mode is:' $APP_ENV;
      "
```

---

## C2) Start the system

```bash
docker compose up
```

---

## C3) Inspect logs

```bash
docker compose logs app
```

You should see:

```
Mode is:
```

---

## C4) Inspect container environment

```bash
docker compose exec app env
```

Look carefully:

* Is `APP_ENV` defined?

❌ No.

---

## Step 5 — Inspect rendered config

```bash
docker compose config
```

You should see:

* `APP_MODE` is set
* `APP_ENV` is not

---

## Explicit conclusion

> Compose passed exactly what you defined.
>
> The failure is:
>
> * variable name mismatch
> * not missing interpolation
> * not a Docker bug

Fix:

```yaml
APP_ENV: production
```

Restart and verify.

---

# Part D — Debugging Volumes (Data “Disappeared”)

> Goal: Prove whether data loss is real or expected.

---

## D1) Replace Compose file

```yaml
services:
  app:
    image: busybox
    volumes:
      - data:/data
    command: >
      sh -c "
      echo 'hello' > /data/file.txt;
      sleep 3600
      "

volumes:
  data:
```

---

## D2) Start and verify

```bash
docker compose up -d
docker compose exec app cat /data/file.txt
```

✅ You should see:

```
hello
```

---

## D3) Remove containers only

```bash
docker compose down
docker compose up -d
docker compose exec app cat /data/file.txt
```

✅ File still exists.

---

## D4) Remove containers AND volumes

```bash
docker compose down -v
docker compose up -d
docker compose exec app ls /data
```

❌ File is gone.

---

## Explicit conclusion

> Data loss was **intentional**.
>
> The command `down -v` explicitly destroys volumes.
>
> This is not a bug. It is **explicit lifecycle control**.

---

# Final Mental Model (Read This Carefully)

Professional debugging means:

* You do not guess
* You do not restart blindly
* You do not “try things”

You:

1. Observe state
2. Read logs
3. Inspect reality
4. Prove the cause
5. Apply the minimal fix

---

## You fully mastered this lab if you can explain:

1. How to debug wrong hostnames
2. How to debug port confusion
3. How to debug missing environment variables
4. How to debug volume-related data loss
5. Why `docker compose config` is a critical tool

---
