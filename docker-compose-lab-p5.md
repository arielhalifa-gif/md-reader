
# Lab 5 — Environment Variables & Configuration  
**Why Configuration Is Not Code (and Why Compose Owns It)**

**CLI-first | Mentor-guided | Explicit cause → effect | Windows + Linux/macOS**

> Hi 👋  
> In this lab, you will learn something subtle but extremely important:
>
> 👉 **Configuration is part of the system design, not the application code.**
>
> You already know how to *use* environment variables.  
> Now you will understand **why Compose puts configuration where it does**  
> and **what problems this design solves**.
>
> This lab will deliberately show you:
> - what breaks when configuration is hardcoded
> - why `.env` exists
> - how Compose enables the same system to run in different contexts

---

## What you will learn (explicit outcomes)

By the end of this lab, you will be able to explain:

1. Why configuration must live **outside images**
2. Why environment variables are the correct abstraction
3. Why Compose treats configuration as **system-level information**
4. How `.env` files work and when they are loaded
5. How variable interpolation works in Compose
6. How configuration enables reuse, portability, and safety

---

## Prerequisites

- You completed **Labs 1–4**
- You understand:
  - services and volumes
  - Compose lifecycle
  - container recreation
- You are comfortable with:
  - `docker compose up`
  - `docker compose config`
  - `docker inspect`

---

## Critical mindset for this lab

Read this carefully:

> If changing an environment value requires rebuilding an image,  
> **your system design is wrong**.

Images are for **code**.  
Environment variables are for **behavior**.

---

# Part A — Hardcoding configuration (and why it’s a trap)

> Goal: Feel why hardcoded configuration creates fragile systems.

---

## A1) Create a new lab folder

### Linux/macOS
```bash
mkdir compose-lab-5 && cd compose-lab-5
````

### Windows PowerShell

```powershell
mkdir compose-lab-5
cd compose-lab-5
```

---

## A2) Create a Compose file with hardcoded values

Create `compose.yaml`:

### Linux/macOS (bash/zsh)

```bash
cat > compose.yaml <<'YAML'
services:
  app:
    image: busybox
    command: >
      sh -c "
      echo 'Running on port 8080';
      echo 'Environment: development';
      sleep 3600
      "
YAML
```

### Windows PowerShell

```powershell
@"
services:
  app:
    image: busybox
    command: >
      sh -c "
      echo 'Running on port 8080';
      echo 'Environment: development';
      sleep 3600
      "
"@ | Out-File -Encoding utf8 compose.yaml
```

---

## A3) Start the service

```bash
docker compose up -d
docker compose logs app
```

✅ **You should see**

```
Running on port 8080
Environment: development
```

---

## A4) The first problem (change request)

Now imagine this requirement:

> “Run the same system in **production mode** on **port 80**.”

Ask yourself honestly:

* What do you need to change?
* Where do you change it?
* How many places will eventually depend on this value?

Right now, the answer is:

* edit the Compose file
* modify command logic
* risk breaking behavior

This does not scale.

---

## Explicit conclusion

> Hardcoded configuration mixes:
>
> * **system behavior**
> * with **execution logic**
>
> This creates brittle systems.

---

# Part B — Move configuration to environment variables

> Goal: Separate **what the system does** from **how it is executed**.

---

## B1) Replace hardcoded values with environment variables

Edit `compose.yaml`:

```yaml
services:
  app:
    image: busybox
    environment:
      APP_PORT: 8080
      APP_ENV: development
    command: >
      sh -c "
      echo 'Running on port '$APP_PORT;
      echo 'Environment: '$APP_ENV;
      sleep 3600
      "
```

---

## B2) Restart and verify

```bash
docker compose down
docker compose up -d
docker compose logs app
```

✅ **You should see**

```
Running on port 8080
Environment: development
```

---

## Explicit conclusion

> The application logic did not change.
>
> Only configuration changed.
>
> This is correct separation of concerns.

---

# Part C — Change behavior without changing the image

> Goal: Prove that configuration can change **without rebuilding**.

---

## C1) Change values in Compose only

Edit `compose.yaml`:

```yaml
environment:
  APP_PORT: 80
  APP_ENV: production
```

---

## C2) Restart and observe

```bash
docker compose down
docker compose up -d
docker compose logs app
```

✅ **You should see**

```
Running on port 80
Environment: production
```

---

## Explicit conclusion (critical)

> The image stayed the same.
>
> The container behavior changed.
>
> This is exactly what environment variables are for.

---

# Part D — Extract configuration into a `.env` file

> Goal: Remove environment-specific values from the Compose file entirely.

---

## D1) Create a `.env` file

Create `.env` in the same folder:

### Linux/macOS

```bash
cat > .env <<'ENV'
APP_PORT=8080
APP_ENV=development
ENV
```

### Windows PowerShell

```powershell
@"
APP_PORT=8080
APP_ENV=development
"@ | Out-File -Encoding utf8 .env
```

---

## D2) Update `compose.yaml` to use interpolation

Edit `compose.yaml`:

```yaml
services:
  app:
    image: busybox
    environment:
      APP_PORT: ${APP_PORT}
      APP_ENV: ${APP_ENV}
    command: >
      sh -c "
      echo 'Running on port '$APP_PORT;
      echo 'Environment: '$APP_ENV;
      sleep 3600
      "
```

---

## D3) Inspect rendered configuration (very important)

```bash
docker compose config
```

✅ **You should see**

```yaml
APP_PORT: "8080"
APP_ENV: development
```

---

## Explicit conclusion

> Compose loads `.env` automatically.
>
> It interpolates variables **before** creating containers.
>
> `.env` defines **system configuration**, not container internals.

---

# Part E — Change environment by swapping `.env`

> Goal: Run the same system in a different context with zero code changes.

---

## E1) Create a production environment file

```bash
cp .env .env.prod
```

Edit `.env.prod`:

```env
APP_PORT=80
APP_ENV=production
```

---

## E2) Run with alternate environment

### Linux/macOS

```bash
export $(cat .env.prod | xargs)
docker compose up -d
```

### Windows PowerShell

```powershell
Get-Content .env.prod | ForEach-Object {
  $pair = $_.Split('=')
  Set-Item -Path env:$($pair[0]) -Value $pair[1]
}
docker compose up -d
```

Check logs:

```bash
docker compose logs app
```

✅ **You should see**

```
Running on port 80
Environment: production
```

---

## Explicit conclusion

> Same Compose file.
>
> Same image.
>
> Different behavior.
>
> This is how systems move between environments safely.

---

# Part F — Why Compose owns configuration (big picture)

Read this carefully.

> Configuration is:
>
> * shared across services
> * environment-specific
> * changeable without rebuild
>
> Therefore, configuration belongs to:
>
> 👉 **the system definition (Compose)**

Images remain reusable.
Compose adapts behavior.

---

# Final conclusions (state them explicitly)

You fully understood this lab if you can explain:

1. Hardcoding configuration creates brittle systems
2. Environment variables decouple behavior from code
3. Images should not encode environment-specific values
4. `.env` files represent system configuration
5. Compose interpolates configuration deterministically
6. Configuration enables portability and reuse

---
