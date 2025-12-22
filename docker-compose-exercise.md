# Docker Compose Step-by-Step Exercise

## Overview
This exercise will guide you through creating a multi-container application using Docker Compose. You'll build a web application stack with a frontend and backend API, learning all essential Docker Compose concepts along the way.

## Prerequisites
- Docker and Docker Compose installed on your system
- Basic understanding of containers
- Text editor of your choice

---

## Part 1: Understanding Docker Compose Basics

### What is Docker Compose?
Docker Compose is a tool for defining and running multi-container Docker applications. You use a YAML file to configure your application's services, networks, and volumes.

### Basic Commands Reference
```bash
# Start services defined in docker-compose.yml
docker-compose up

# Start services in detached mode (background)
docker-compose up -d

# Stop and remove containers, networks
docker-compose down

# Stop and remove containers, networks, volumes
docker-compose down -v

# View running services
docker-compose ps

# View logs
docker-compose logs

# View logs for specific service
docker-compose logs <service-name>

# Execute command in running container
docker-compose exec <service-name> <command>

# Build or rebuild services
docker-compose build

# Start existing containers
docker-compose start

# Stop running containers without removing
docker-compose stop

# Restart services
docker-compose restart
```

---

## Part 2: Creating Your First Docker Compose File

### Exercise Setup
Create a new directory for this exercise:
```bash
mkdir docker-compose-exercise
cd docker-compose-exercise
```

### Step 1: Simple Single Service (nginx)

Create a file named `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    container_name: my_nginx_server
    ports:
      - "8080:80"
```

**Analysis:**
- `version`: Specifies Docker Compose file format version
- `services`: Defines the containers to run
- `web`: Service name (you choose this)
- `image`: Docker image to use (nginx:alpine - lightweight nginx)
- `container_name`: Custom name for the container (optional)
- `ports`: Maps host port 8080 to container port 80

**Try it:**
```bash
docker-compose up -d
```

Visit `http://localhost:8080` in your browser. You should see the nginx welcome page.

**Clean up:**
```bash
docker-compose down
```

---

## Part 3: Adding Volumes

### Step 2: Serving Custom Content

Create an `index.html` file in your exercise directory:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Docker Compose Exercise</title>
</head>
<body>
    <h1>Hello from Docker Compose!</h1>
    <p>This content is served from a volume.</p>
</body>
</html>
```

Update your `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    container_name: my_nginx_server
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
```

**Analysis of Volumes:**
- `./index.html`: Path on host machine (relative to docker-compose.yml)
- `/usr/share/nginx/html/index.html`: Path inside container
- `:ro`: Read-only flag (optional, prevents container from modifying host file)

**Types of Volumes:**
1. **Bind mounts** (what we used): `./host/path:/container/path`
2. **Named volumes**: `volume_name:/container/path`
3. **Anonymous volumes**: `/container/path`

**Try it:**
```bash
docker-compose up -d
```

Visit `http://localhost:8080` - you should see your custom HTML!

Try editing `index.html` while the container is running and refresh the browser.

---

## Part 4: Multi-Container Application (Default Network)

### Step 3: Adding a Backend API

In this step, we'll add an API service that needs to communicate with the web frontend. Docker Compose will automatically create a default network for them.

Create this directory structure:
```
docker-compose-exercise/
├── docker-compose.yml
├── index.html
└── api/
    ├── app.py
    ├── requirements.txt
    └── Dockerfile
```

Create `api/app.py`:

```python
from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn
from datetime import datetime

app = FastAPI(title="Docker Compose Exercise API")

class MessageResponse(BaseModel):
    message: str
    timestamp: str
    environment: str

class HealthResponse(BaseModel):
    status: str
    api_version: str

@app.get("/", response_model=MessageResponse)
async def root():
    return {
        "message": "Hello from FastAPI!",
        "timestamp": datetime.now().isoformat(),
        "environment": "Docker Compose"
    }

@app.get("/health", response_model=HealthResponse)
async def health():
    return {
        "status": "healthy",
        "api_version": "1.0.0"
    }

@app.get("/data")
async def get_data():
    """Simulated data endpoint"""
    return {
        "users": ["Alice", "Bob", "Charlie"],
        "count": 3,
        "source": "In-memory data"
    }

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Create `api/requirements.txt`:

```
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
```

Create `api/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Expose port
EXPOSE 8000

# Run with uvicorn
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Update `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # Frontend nginx server
  web:
    image: nginx:alpine
    container_name: frontend_server
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
    depends_on:
      - api

  # Backend FastAPI
  api:
    build: ./api
    container_name: backend_api
    ports:
      - "8000:8000"
```

**Analysis - Default Network Behavior:**

Notice we **didn't define any networks** in this configuration! Docker Compose automatically:

1. **Creates a default network** named `<directory>_default` (e.g., `docker-compose-exercise_default`)
2. **Connects all services** to this default bridge network
3. **Enables automatic DNS resolution** - services can reach each other using service names as hostnames

**Key Points:**
- The `web` service can connect to `api` using `api:8000` as the hostname
- Both services are on the same network automatically
- No explicit network configuration needed for basic communication!

### Images
- **Using pre-built images**: `nginx:alpine`
  - Format: `image_name:tag`
  - Alpine variants are smaller, faster to download
- **Building custom images**: `build: ./api`
  - Docker Compose builds image from Dockerfile in specified directory
  - Image will be named `<project>_api`
  - First build takes longer, subsequent builds use cache

### Container Names
- Without `container_name`: Docker generates names like `project_service_1`
- With `container_name`: You control the exact name
- **Benefit**: Easier to reference and identify containers
- Example: `backend_api` instead of `docker-compose-exercise_api_1`

### Ports
- Format: `"HOST_PORT:CONTAINER_PORT"`
- `"8080:80"`: Access container port 80 via host port 8080
- `"8000:8000"`: API accessible on port 8000 on both host and container
- **Important**: Only expose ports that need external access from your host machine

### Volumes
**Bind mount** (`./index.html`):
- Links directly to host filesystem
- Changes reflect immediately in the container
- Good for development
- **Use for**: Source code, configuration files during development
- Format: `./host/path:/container/path:ro` (`:ro` = read-only)

### Depends_on
- `depends_on` controls startup order
- `web` waits for `api` container to start
- **Important**: Only waits for container to start, NOT for application to be ready!
- Startup order: `api` → `web`
- Doesn't guarantee the API is ready to accept requests when web starts

---

## Part 5: Running and Testing the Default Network

### Step 4: Launch the Stack and Explore Networking

```bash
# Build and start all services
docker-compose up --build -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

**Explore the Default Network:**

```bash
# List Docker networks - you'll see the default network created
docker network ls

# You should see something like: docker-compose-exercise_default

# Inspect the default network to see all connected containers
docker network inspect docker-compose-exercise_default

# Look for the "Containers" section in the output
```

The output shows both containers connected to the same network with their IP addresses!

### Step 5: Testing Service Communication

```bash
# Test the API directly
curl http://localhost:8000/

# Test the health endpoint
curl http://localhost:8000/health

# Test the data endpoint
curl http://localhost:8000/data

# View API documentation (FastAPI automatic docs!)
# Open in browser: http://localhost:8000/docs

# Test the web server
curl http://localhost:8080/
```

### Step 6: Understanding DNS Resolution on Default Network

The magic happens through Docker's built-in DNS server:

```bash
# Enter the API container
docker-compose exec api sh

# Inside the container, try these commands:

# 1. Check if we can resolve the web service
nslookup web

# 2. Try pinging the web service by name
ping -c 3 web

# 3. Check what network interface we have
ip addr show

# Exit the container
exit
```

**What's happening:**
- Docker Compose's default network provides automatic DNS
- Service name `web` resolves to the web container's IP
- Service name `api` resolves to the API container's IP
- No hardcoded IPs needed!
- Services discover each other automatically

**Key Observation:**
```bash
# From your host machine
ping web  # This will FAIL - DNS only works inside the Docker network

# But from inside a container
docker-compose exec api ping -c 2 web  # This WORKS!
```

---

## Part 6: Custom Networks for Better Control

### Step 7: Adding a Custom Network

Now let's see why you might want to define custom networks. Update your `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # Frontend nginx server
  web:
    image: nginx:alpine
    container_name: frontend_server
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - app-network
    depends_on:
      - api

  # Backend FastAPI
  api:
    build: ./api
    container_name: backend_api
    ports:
      - "8000:8000"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**What Changed:**
- Added `networks:` section at the top level to define `app-network`
- Added `networks: - app-network` to each service
- Explicitly named our network instead of using the default

**Why Use Custom Networks:**

1. **Explicit Control**: You choose the network name
2. **Multiple Networks**: Can create separate networks for different service groups
3. **Network Isolation**: Services on different networks can't communicate
4. **Better Documentation**: Named networks make architecture clearer
5. **Advanced Configuration**: Can set custom network settings (IP ranges, drivers, etc.)

**Restart with the new configuration:**

```bash
# Stop and remove old containers
docker-compose down

# Start with new network configuration
docker-compose up -d

# List networks - now you'll see 'docker-compose-exercise_app-network'
docker network ls

# Inspect your custom network
docker network inspect docker-compose-exercise_app-network
```

**Test that everything still works:**

```bash
# Test the API still works
curl http://localhost:8000/data

# Enter API container and verify network connectivity
docker-compose exec api ping -c 2 web
```

Communication works exactly the same! The difference is organizational.

### Step 8: Multiple Networks Example

Here's a use case showing how you might separate concerns with multiple networks. Let's imagine we add a Redis cache:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    container_name: frontend_server
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - frontend-network
    depends_on:
      - api

  api:
    build: ./api
    container_name: backend_api
    ports:
      - "8000:8000"
    networks:
      - frontend-network  # API is on BOTH networks
      - backend-network   # This allows it to bridge between them
    depends_on:
      - cache

  cache:
    image: redis:alpine
    container_name: redis_cache
    networks:
      - backend-network   # Cache only on backend network
    # Note: No ports exposed! Only API can access it via internal network

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
```

**Analysis - Multi-Network Architecture:**

```
┌─────────────────┐
│   web (nginx)   │  frontend-network
└────────┬────────┘
         │
         │ Can communicate
         ↓
┌─────────────────┐
│   api (FastAPI) │  frontend-network + backend-network
└────────┬────────┘
         │
         │ Can communicate
         ↓
┌─────────────────┐
│  cache (Redis)  │  backend-network only
└─────────────────┘
```

**Security Benefits:**
- `web` **cannot** directly access `cache` (they're on different networks)
- `cache` has no exposed ports, only accessible via `backend-network`
- `api` acts as a gateway between frontend and backend
- Each layer is isolated from the others

**Test the isolation:**

```bash
# You can test this if you want to see isolation in action
# docker-compose down
# docker-compose up -d

# Try to ping cache from web container - it will FAIL!
# docker-compose exec web ping -c 2 cache
# Error: cache: Name or service not known

# But API can still reach cache
# docker-compose exec api ping -c 2 cache
# Success!
```

---

## Part 7: Network Comparison Summary

### Default Network vs Custom Network

| Feature | Default Network | Custom Network |
|---------|----------------|----------------|
| **Creation** | Automatic | Explicit definition required |
| **Name** | `<project>_default` | Your choice (e.g., `app-network`) |
| **DNS Resolution** | ✅ Yes | ✅ Yes |
| **Isolation** | All services together | Can separate with multiple networks |
| **Configuration** | No control | Full control (IP ranges, drivers, etc.) |
| **Best for** | Simple apps, quick prototyping | Production, complex architectures |

### When to Use What:

**Use Default Network when:**
- Quick development/testing
- All services need to communicate
- Simple single-stack application
- You don't need network isolation

**Use Custom Networks when:**
- Need network isolation (frontend/backend separation)
- Multiple applications on same host
- Want explicit network documentation
- Need specific network configuration
- Production deployments

---

## Part 8: Environment Variables and Configuration

### Step 9: Environment Variables and Env Files

Create `.env` file:

```env
API_PORT=8000
WEB_PORT=8080
ENVIRONMENT=development
```

Update `docker-compose.yml` to use variables:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    container_name: frontend_server
    ports:
      - "${WEB_PORT}:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - app-network
    depends_on:
      - api

  api:
    build: ./api
    container_name: backend_api
    ports:
      - "${API_PORT}:8000"
    environment:
      - ENVIRONMENT=${ENVIRONMENT}
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**Analysis of Environment Variables:**
- Variables in `.env` are automatically loaded
- Reference with `${VARIABLE_NAME}`
- Can be overridden with shell environment variables
- Keeps sensitive data out of version control (add `.env` to `.gitignore`)

---

## Part 9: Volume Management Deep Dive

### Step 10: Understanding Different Volume Types

Let's explore volumes in more detail. We'll add a named volume for logs.

Update your `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    container_name: frontend_server
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
      - nginx-logs:/var/log/nginx
    networks:
      - app-network
    depends_on:
      - api

  api:
    build: ./api
    container_name: backend_api
    ports:
      - "8000:8000"
    volumes:
      - ./api:/app:ro
      - api-logs:/var/log
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  nginx-logs:
  api-logs:
```

**Analysis of Volume Types:**

1. **Bind Mount** (`./index.html:/usr/share/nginx/html/index.html:ro`):
   - Direct link to host file system
   - Changes on host immediately reflect in container
   - Perfect for development

2. **Named Volume** (`nginx-logs:`, `api-logs:`):
   - Managed by Docker
   - Data persists between container restarts
   - Stored in Docker's managed storage area

```bash
# List all volumes
docker volume ls

# You should see: 
# docker-compose-exercise_nginx-logs
# docker-compose-exercise_api-logs

# Inspect a named volume
docker volume inspect docker-compose-exercise_nginx-logs

# See volume contents
docker-compose exec web ls -la /var/log/nginx
```

### Volume Cleanup

```bash
# Remove containers but keep volumes
docker-compose down

# Volumes still exist
docker volume ls

# Start again - volume data persists!
docker-compose up -d

# Remove everything including volumes
docker-compose down -v

# Now volumes are gone
docker volume ls
```

---

## Part 10: Troubleshooting Commands

### Useful Debugging Commands

```bash
# View logs for all services
docker-compose logs

# Follow logs in real-time
docker-compose logs -f

# View logs for specific service
docker-compose logs api

# View last 50 lines
docker-compose logs --tail=50

# Check running processes
docker-compose top

# View resource usage
docker stats $(docker-compose ps -q)

# Rebuild a specific service
docker-compose build api

# Restart a single service
docker-compose restart api

# Stop a single service
docker-compose stop web

# Remove a specific service
docker-compose rm -s -v api
```

---

## Part 11: Practice Exercises

### Exercise A: Add Redis Cache
Add a Redis container to your stack:
- Use `redis:alpine` image
- Name it `cache`
- Expose port 6379 (optional, for external access)
- Add it to the `app-network`
- Make the API depend on it
- Don't expose the port if you want it internal-only

**Bonus**: Update the API code to connect to Redis and cache some data!

### Exercise B: Add Health Checks
Add health checks to your services:

```yaml
services:
  api:
    # ... existing config ...
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Exercise C: Add Another API Service
Create a second API service:
- Build from the same `./api` directory
- Name it `api-v2`
- Run on port 8001
- Add it to the network
- Test that both APIs work independently

---

## Part 12: Best Practices Summary

### 1. Container Naming
- Use descriptive `container_name` for important services
- Follow naming convention: `project_service_environment`

### 2. Port Management
- Only expose necessary ports
- Use non-standard ports to avoid conflicts
- Document all exposed ports

### 3. Volume Strategy
- Use named volumes for persistent data (logs, uploads, cache data)
- Use bind mounts for development code
- Always backup important volumes
- Add volume removal protection for production
- Use `:ro` flag when containers only need read access

### 4. Network Design
- Create separate networks for different tiers
- Use custom networks instead of default
- Document service communication patterns

### 5. Dependency Management
- Use `depends_on` for startup order
- Consider using health checks for readiness
- Document service dependencies

### 6. Environment Variables
- Use `.env` files for configuration
- Never commit sensitive data
- Document all required variables
- Use different .env files for different environments

### 7. Image Selection
- Prefer official images
- Use specific tags, not `latest`
- Use Alpine variants for smaller size
- Regularly update base images

---

## Final Docker Compose Cheat Sheet

```bash
# Lifecycle Commands
docker-compose up              # Start services
docker-compose up -d           # Start in background
docker-compose down            # Stop and remove containers
docker-compose down -v         # Also remove volumes
docker-compose restart         # Restart all services
docker-compose pause           # Pause services
docker-compose unpause         # Unpause services

# Building
docker-compose build           # Build all images
docker-compose build --no-cache # Build without cache
docker-compose pull            # Pull all images

# Inspection
docker-compose ps              # List containers
docker-compose top             # Display running processes
docker-compose logs            # View logs
docker-compose logs -f         # Follow logs
docker-compose config          # Validate and view config

# Execution
docker-compose exec <service> <command>  # Run command
docker-compose run <service> <command>   # Run one-off command

# Scaling
docker-compose up -d --scale api=3      # Run 3 instances of api
```

---

## Cleanup

When you're done with the exercise:

```bash
# Stop and remove everything
docker-compose down -v

# Remove orphaned images
docker image prune

# Remove all unused resources
docker system prune -a
```

---

## Summary

You've learned:
- ✅ Basic Docker Compose commands
- ✅ How to use pre-built images and build custom images
- ✅ Container naming conventions
- ✅ Port mapping for external access
- ✅ Volume types (bind mounts and named volumes)
- ✅ Default vs custom network creation
- ✅ Network isolation with multiple networks
- ✅ Service dependencies with `depends_on`
- ✅ Environment variable management
- ✅ Troubleshooting and debugging techniques

## Next Steps
- Explore health checks and readiness probes
- Learn about Docker Compose override files
- Study production deployment patterns
- Practice with more complex multi-service applications
- Explore Docker Swarm or Kubernetes for orchestration

---

**Congratulations!** You've completed the Docker Compose exercise. Keep practicing by building your own multi-container applications!
