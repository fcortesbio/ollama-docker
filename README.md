# Docker Compose for Ollama with Traefik
🐳 Ollama Load-Balanced Service for IDE development

This repository contains the configuration to run a scalable, high-concurrency Ollama API server locally using **Docker Compose** and **Traefik** as a load balancer.

This setup is optimized for local development environments (like Cursor or Antigravity) running on capable machines (tunned for my laptop with Intel i5-12500H with 40GB RAM) to maximize throughput and utilize integrated Intel graphics acceleration.

--- 

## 🚀 Architecture and Goals

| Component | Role | Notes |
| :--- | :--- | :--- |
| **Ollama** | LLM Inference Server | Runs **3 replicas** of the LLM (e.g., Llama 3 8B). |
| **Traefik** | Reverse Proxy & Load Balancer | Routes traffic from a single endpoint to the least busy Ollama replica, using the **Least Connection** algorithm. |
| **Intel iGPU** | Acceleration | Uses the `OLLAMA_ACCELERATE_SYSTEM=1` flag and exposes the `/dev/dri` device for integrated graphics acceleration (Iris Xe). |
| **Access** | Unified Endpoint | Accessible via **`http://ollama.localhost`** or **`http://localhost`**. |

---

## 📋 Prerequisites

Before starting, ensure your system meets these requirements:

1.  **Docker & Docker Compose:** Installed and running on your Arch Linux machine.
2.  **Intel Graphics Drivers:** The necessary drivers must be installed to expose the `/dev/dri` device for iGPU acceleration.
3.  **Sufficient RAM:** This configuration is optimized for a machine with **32 GB+ of RAM** (target model size $\times 3$).

---

## 🛠️ Deployment Steps

### 1. File Structure

Ensure you have the following files in your repository root:

* `docker-compose.yml` (Included below)
* `README.md` (This file)

### 2. Deploy the Stack

Navigate to the directory containing `docker-compose.yml` and run:

```bash
docker compose up -d
```

This command will:

    - Pull the traefik:v3.6 and ollama/ollama:latest images.

    - Create the necessary Docker network and volume (ollama_data).

    - Start one Traefik container.

    - Start three Ollama replicas, configured for iGPU acceleration and remote access.

### 3. Install the Model
Once the containers are running, you must download the desired model into the shared volume. We are using Llama 3 8B as the target model.

Run the following command to execute the ollama pull command inside one of the running Ollama containers:

```bash
# This uses the service name 'ollama' and executes the pull command
docker compose exec ollama ollama pull llama3:8b-instruct-q4_0
```

---

## 🔌 Connection Details for IDEs

To connect your IDE (Cursor, Antigravity, etc.) to the load-balanced Ollama service, use one of the following unified endpoints:

| Connection Method | URL | Notes |
| :--- | :--- | :--- |
| **Recommended (Hostname)** | `http://ollama.localhost:11480` | Uses the Traefik hostname router for a clean URL. |
| **Alternative (Direct Port)** | `http://localhost:11480` | Uses the host's mapped HTTP port (11480 → Traefik's internal 80). |

Traefik will handle the routing and load balancing of all API calls (`/api/generate`, `/api/chat`, etc.) across the three available Ollama instances.

---

## ⚙️ Configuration Notes

### Load Balancing Algorithm

The current configuration uses the **Least Connection** load balancing algorithm, which is superior for long-running LLM inference tasks. This algorithm routes requests to the replica with the fewest active connections.

This is configured by the following label in `docker-compose.yml`:

```yaml
- "traefik.http.services.ollama-service.loadbalancer.algorithm=leastconn"
```

If you wanted to revert to the default **Round Robin** algorithm, simply remove this line.

### Port Mapping Summary

To maintain good neighborhood (avoid conflicts with common services), the host ports were mapped as follows:

- **Host Port 11480** → Traefik HTTP Entrypoint (80)
- **Host Port 11481** → Traefik Dashboard (8080)

These non-standard ports prevent conflicts with other services that might use ports 80 and 8080.

### Health Checks

Traefik performs health checks on each Ollama replica every **10 seconds** using the `/api/tags` endpoint. Unhealthy replicas are automatically removed from the load balancing pool until they recover.

---

## 🚦 Monitoring and Troubleshooting

### Traefik Dashboard

You can monitor Traefik's routing and the health of your Ollama replicas by accessing the dashboard:

**Traefik Dashboard:** `http://localhost:11481`

The dashboard displays:
- Active routers and services
- Load balancer status for each Ollama replica
- Health check results
- Real-time request metrics

### Checking Container Status

To verify all containers are running:

```bash
docker compose ps
```

### Viewing Logs

To troubleshoot issues, check the logs:

```bash
# View all logs
docker compose logs

# Follow logs in real-time
docker compose logs -f

# View logs for specific service
docker compose logs ollama
docker compose logs traefik
```

### Common Issues

1. **Replicas not starting:** Check available RAM and ensure your system meets the prerequisites.
2. **Health checks failing:** Verify the model is downloaded and Ollama is responding to `/api/tags`.
3. **Hostname resolution issues:** Add `127.0.0.1 ollama.localhost` to `/etc/hosts` if needed.

---

## 🎯 Testing the Setup

Once everything is running and the model is downloaded, test the endpoint:

```bash
# Test via hostname
curl -s http://ollama.localhost:11480/api/generate -d '{
  "model": "llama3:8b-instruct-q4_0",
  "prompt": "Say hello in one sentence.",
  "stream": false
}' | jq -r '.response'

# Or via direct port
curl -s http://localhost:11480/api/generate -d '{
  "model": "llama3:8b-instruct-q4_0",
  "prompt": "Say hello in one sentence.",
  "stream": false
}' | jq -r '.response'
```

---

## 🛑 Stopping the Stack

To stop and remove all containers:

```bash
docker compose down
```

To stop and remove containers **and volumes** (deletes downloaded models):

```bash
docker compose down -v
```

---

## 📝 Notes

- **Docker Compose v2+ required:** This configuration uses the `docker compose` command (not `docker-compose`).
- **Swarm mode not required:** The `deploy.replicas` directive works with Docker Compose v2+ without Swarm.
- **iGPU acceleration:** Requires proper Intel graphics drivers and `/dev/dri` device access.
- **Model persistence:** Models are stored in the `ollama_data` Docker volume and persist across container restarts.

---

Enjoy your load-balanced local LLM setup! 🚀

