# Guide to the Local-LLM Docker Compose Stack
This guide explains the contents of your `docker-compose.yml` and `.env` files, line by line, to help you understand how the stack is assembled. It also includes context about why certain directives are used and how the containers communicate. Use this document as a reference for your own README.
## Table of contents
1. [Overview](#overview)
2. [Top-level project metadata](#top-level-project-metadata)
3. [Services](#services)
1. [`vllm-chat` – Chat API server](#vllm-chat-–-chat-api-server)
2. [`vllm-embed` – Embeddings API server](#vllm-embed-–-embeddings-api-server)
3. [`openwebui` – Web UI](#openwebui-–-web-ui)
4. [`qdrant` – Vector database](#qdrant-–-vector-database)
5. [`minio` – Object storage](#minio-–-object-storage)
4. [Networking and volumes](#networking-and-volumes)
5. [`.env` variables and parameter substitution](#env-variables-and-parameter-substitution)
6. [Data flow summary](#data-flow-summary)
## Overview
This Compose stack deploys a **local LLM chat and embedding service** using [vLLM](https://github.com/vllm-project/vllm), a [Qdrant](https://qdrant.tech/) vector store, an [Open WebUI](https://github.com/open-webui/open-webui/) front-end, and optional [MinIO](https://min.io/) object storage. Each service runs in its own container but shares a common Docker network so they can address each other by name. Named volumes persist data such as cached models, vector collections and UI state between container restarts. The configuration is controlled via environment variables defined in a `.env` file.
## Top-level project metadata
```yaml
version: "3.9"
name: local-llm-vllm-rtx3080
```
| Key | Meaning |
| --- | --- |
| `version: "3.9"` | Specifies the Compose file format version. Version 3.9 works with modern Docker Compose and supports features like GPU scheduling. |
| `name: local-llm-vllm-rtx3080` | Sets the **project name**. Docker prefixes containers, networks and volumes with this value, keeping resources for this stack grouped under a common namespace (e.g., `local-llm-vllm-rtx3080_ai-net`). |
## Services
### `vllm-chat` – Chat API server
This service exposes the large-language-model (LLM) chat API. It runs the [vLLM OpenAI-compatible server](https://github.com/vllm-project/vllm) inside a container and uses your GPU for acceleration.
```yaml
vllm-chat:
image: vllm/vllm-openai:latest
restart: unless-stopped
environment:
- NVIDIA_VISIBLE_DEVICES=all
- HUGGING_FACE_HUB_TOKEN=${HUGGING_FACE_HUB_TOKEN}
command:
- "--model=${VLLM_CHAT_MODEL:-Qwen/Qwen2.5-3B-Instruct}"
- "--dtype=${VLLM_CHAT_DTYPE:-auto}"
- "--gpu-memory-utilization=${VLLM_CHAT_GPU_UTILIZATION:-0.85}"
- "--port=8000"
- "--trust-remote-code"
volumes:
- hf_cache:/root/.cache/huggingface
deploy:
resources:
reservations:
devices:
- driver: nvidia
count: all
capabilities: ["gpu"]
networks:
- ai-net
```
**Explanation**
* **`image: vllm/vllm-openai:latest`** – Uses the official vLLM Docker image that exposes an OpenAI-style API. The `latest` tag pulls the most recent release.
* **`restart: unless-stopped`** – Ensures the container automatically restarts on failure or host reboot unless you explicitly stop it.
* **`environment`** – Injects environment variables into the container:
* `NVIDIA_VISIBLE_DEVICES=all` exposes all GPUs to the container. vLLM uses CUDA and needs GPU access.
* `HUGGING_FACE_HUB_TOKEN` passes your Hugging Face token from `.env` to authenticate model downloads.
* **`command`** – Overrides the default entrypoint with arguments to configure the server:
* `--model=${VLLM_CHAT_MODEL:-Qwen/Qwen2.5-3B-Instruct}` selects which chat model to load. If the environment variable `VLLM_CHAT_MODEL` is undefined, it defaults to `Qwen/Qwen2.5-3B-Instruct`.
* `--dtype=${VLLM_CHAT_DTYPE:-auto}` controls numerical precision (e.g. `auto`, `float16`).
* `--gpu-memory-utilization=${VLLM_CHAT_GPU_UTILIZATION:-0.85}` sets how aggressively vLLM uses GPU memory (0–1). Lower values reduce memory usage but may degrade throughput.
* `--port=8000` exposes the API on port 8000 inside the Docker network.
* `--trust-remote-code` allows vLLM to execute model-specific code downloaded from repositories. This is required for some models but carries a security risk.
* **`volumes`** – Mounts the named volume `hf_cache` at `/root/.cache/huggingface`. This caches downloaded model weights and tokenizers so subsequent container restarts reuse the same files.
* **`deploy.resources.reservations.devices`** – Requests GPUs from Docker’s [NVIDIA runtime](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/user-guide.html). `count: all` uses any available GPUs. Without this section, the service might not see your GPU.
* **`networks: ai-net`** – Connects the service to the `ai-net` network, making it discoverable to other containers by the name `vllm-chat`.
### `vllm-embed` – Embeddings API server
This service is nearly identical to `vllm-chat` but serves an embedding model. It listens on port 8001 and uses its own environment variables.
```yaml
vllm-embed:
image: vllm/vllm-openai:latest
restart: unless-stopped
environment:
- NVIDIA_VISIBLE_DEVICES=all
- HUGGING_FACE_HUB_TOKEN=${HUGGING_FACE_HUB_TOKEN}
command:
- "--model=${VLLM_EMBED_MODEL:-nomic-ai/nomic-embed-text-v1}"
- "--dtype=${VLLM_EMBED_DTYPE:-auto}"
- "--gpu-memory-utilization=${VLLM_EMBED_GPU_UTILIZATION:-0.5}"
- "--port=8001"
- "--trust-remote-code"
volumes:
- hf_cache:/root/.cache/huggingface
deploy:
resources:
reservations:
devices:
- driver: nvidia
count: all
capabilities: ["gpu"]
networks:
- ai-net
```
**Differences from `vllm-chat`**
* **Model** – Defaults to `nomic-ai/nomic-embed-text-v1` when `VLLM_EMBED_MODEL` is not set.
* **GPU utilization** – Uses `VLLM_EMBED_GPU_UTILIZATION` (default 0.5) to leave more memory for the chat model.
* **Port** – Listens on 8001. The service will be reachable at `http://vllm-embed:8001` from other containers.
### `openwebui` – Web UI
Open WebUI provides a user interface for chat and retrieval-augmented generation (RAG). It depends on both vLLM services and Qdrant.
```yaml
openwebui:
image: ghcr.io/open-webui/open-webui:main
restart: unless-stopped
depends_on:
- qdrant
- vllm-chat
- vllm-embed
environment:
- OPENAI_API_BASE_URL=http://vllm-chat:8000/v1
- OPENAI_API_KEY=local-key
- VECTOR_DB=qdrant
- QDRANT_URI=${QDRANT_URI:-http://qdrant:6333}
- WEBUI_NAME=${WEBUI_NAME:-Local LLM (vLLM on 3080)}
volumes:
- openwebui_data:/app/backend/data
- openwebui_cache:/root/.cache
ports:
- "0.0.0.0:3000:8080"
networks:
- ai-net
```
**Explanation**
* **`image: ghcr.io/open-webui/open-webui:main`** – Pulls the latest main branch of Open WebUI from GitHub Container Registry.
* **`depends_on`** – Instructs Compose to start Qdrant and the two vLLM services first. It doesn’t enforce ordering for network connectivity but avoids race conditions during container creation.
* **`environment`**:
* `OPENAI_API_BASE_URL=http://vllm-chat:8000/v1` points the UI at the chat API on the `ai-net` network. Because Compose sets up service-name DNS on the network, `vllm-chat` resolves to the chat container’s IP address【491764299003153†L851-L885】.
* `OPENAI_API_KEY=local-key` is a placeholder. Open WebUI requires a key even when accessing a local API.
* `VECTOR_DB=qdrant` tells Open WebUI to use Qdrant as its vector store.
* `QDRANT_URI` defaults to `http://qdrant:6333`. You can override this in `.env` if you point to a remote Qdrant.
* `WEBUI_NAME` sets the name shown in the UI. If unset, it defaults to “Local LLM (vLLM on 3080)”.
* **`volumes`** – Two named volumes are mounted:
* `openwebui_data` persists the internal database, user sessions and uploads at `/app/backend/data`.
* `openwebui_cache` caches Python packages and runtime files at `/root/.cache`.
* **`ports: "0.0.0.0:3000:8080"`** – Binds port 8080 inside the container to port 3000 on **all host interfaces**, exposing the UI to your LAN.
* **`networks: ai-net`** – Connects to the shared network so it can reach `vllm-chat`, `vllm-embed` and `qdrant` by service name.
### `qdrant` – Vector database
Qdrant stores embeddings and metadata for retrieval-augmented generation. By default it runs without authentication and stores data in `/qdrant/storage`.
```yaml
qdrant:
image: qdrant/qdrant:latest
restart: unless-stopped
volumes:
- qdrant_storage:/qdrant/storage
healthcheck:
test: ["CMD-SHELL", "(command -v curl >/dev/null 2>&1 && curl -fsS http://localhost:6333/readyz) || (command -v wget >/dev/null 2>&1 && wget -qO- http://localhost:6333/readyz) || exit 1"]
interval: 5s
timeout: 3s
start_period: 30s
retries: 50
networks:
- ai-net
```
**Explanation**
* **`image`** – Uses Qdrant’s official image. A new release may include improved performance or features.
* **`restart: unless-stopped`** – Keeps the database running persistently.
* **`volumes: qdrant_storage:/qdrant/storage`** – Persists vectors and collections on a named volume. According to the Docker docs, volumes are the recommended way to persist data because they are managed by Docker and can be easily backed up or migrated【785634518959600†L876-L885】. A volume’s contents exist outside the lifecycle of a container, so data survive container deletion【785634518959600†L910-L912】.
* **`healthcheck`** – Pings the `/readyz` endpoint every 5 seconds to ensure Qdrant is ready before Open WebUI tries to connect. It uses whichever client (curl or wget) exists in the image. `start_period` gives Qdrant 30 seconds to boot, and `retries: 50` allows up to 50 failures before marking the service as
unhealthy.
* **`networks`** – Connects the database to `ai-net`. Open WebUI and other services reference it via `qdrant`.
### `minio` – Object storage
MinIO provides S3-compatible object storage. It’s optional but useful for storing large files or documents for RAG.
```yaml
minio:
image: quay.io/minio/minio:latest
restart: unless-stopped
environment:
- MINIO_ROOT_USER=${MINIO_ROOT_USER}
- MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
command: server /data --console-address ":9001"
volumes:
- minio_data:/data
ports:
- "127.0.0.1:9000:9000"
- "127.0.0.1:9001:9001"
networks:
- ai-net
```
**Explanation**
* **`environment`** – Supplies the admin access key and secret key from `.env`. MinIO defaults to `minioadmin`/`minioadmin` if not set, but you should set strong credentials in production.
* **`command: server /data --console-address ":9001"`** – Launches the MinIO server with its data directory at `/data` and exposes the web console on port 9001. The S3 API listens on port 9000.
* **`volumes: minio_data:/data`** – Persists all buckets and objects on a named volume.
* **`ports`** – Binds both the API (`9000`) and console (`9001`) **only on localhost** (`127.0.0.1`), preventing external hosts from connecting. You can remove the `127.0.0.1` prefix to expose MinIO to your LAN.
* **`networks`** – Allows other containers on `ai-net` to access MinIO at `http://minio:9000` or `http://minio:9001`. For example, you could configure Open WebUI or vLLM to read documents from MinIO.
## Networking and volumes
### Network
At the bottom of the file the `networks` section defines a single named network:
```yaml
networks:
ai-net: {}
```
Docker Compose automatically sets up a default bridge network for your app. Each service joins that network and becomes reachable by the service name. The Docker documentation explains that by default Compose creates a single network and each container joins it; containers can communicate by referencing each other’s service names【491764299003153†L851-L885】. Naming the network (`ai-net`) makes the intent explicit and prevents Docker from generating a generic name.
### Volumes
The following named volumes are declared:
```yaml
volumes:
openwebui_data:
openwebui_cache:
qdrant_storage:
minio_data:
hf_cache:
```
Volumes are managed by Docker and persist data even when containers are removed. They are the preferred mechanism for storing data created or used by containers【785634518959600†L876-L885】. For this stack:
* **`hf_cache`** – Shared by both vLLM services to cache downloaded models from Hugging Face.
* **`openwebui_data`** – Persists Open WebUI’s SQLite database, attachments and sessions.
* **`openwebui_cache`** – Caches Python packages and other runtime data for Open WebUI.
* **`qdrant_storage`** – Stores Qdrant collections and indexes.
* **`minio_data`** – Stores MinIO buckets and objects.
Because volumes live outside the container lifecycle, your data remains intact when you recreate or update services【785634518959600†L910-L912】.
## `.env` variables and parameter substitution
My stack uses environment variables defined in a `.env` file. Compose automatically loads variables from the file and substitutes them wherever you reference them via `${VAR}` or `${VAR:-default}`. The values can be customised without editing the `docker-compose.yml`. Here is a simplified view of your `.env`:
```dotenv
HUGGING_FACE_HUB_TOKEN=… # Required to download models from Hugging Face
VLLM_CHAT_MODEL=Qwen/Qwen2.5-3B-Instruct
VLLM_CHAT_DTYPE=auto
VLLM_CHAT_GPU_UTILIZATION=0.85
VLLM_EMBED_MODEL=nomic-ai/nomic-embed-text-v1
VLLM_EMBED_DTYPE=auto
VLLM_EMBED_GPU_UTILIZATION=0.50
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=supersecretlongpw
QDRANT_URI=http://qdrant:6333
WEBUI_NAME=Local LLM (vLLM on 3080)
```
The `${VAR:-fallback}` syntax means “use the value of `VAR` if it is defined; otherwise, use `fallback`.” For example, `--model=${VLLM_CHAT_MODEL:-Qwen/Qwen2.5-3B-Instruct}` loads the model specified in `VLLM_CHAT_MODEL` or defaults to Qwen2.5-3B-Instruct.
## Data flow summary
1. **Clients** (browsers or API consumers) connect to Open WebUI via `http://<host>:3000`. The UI sits behind your LAN firewall.
2. Open WebUI sends chat requests to `http://vllm-chat:8000/v1` and embedding requests to `http://vllm-embed:8001/v1` using service-name DNS on `ai-net`【491764299003153†L851-L885】.
3. When retrieving documents, Open WebUI writes and reads vectors from Qdrant at
`http://qdrant:6333`. Qdrant uses persistent storage provided by the `qdrant_storage` volume.
4. Both vLLM services cache models in `hf_cache`, ensuring that model downloads happen once and persist across restarts.
5. MinIO runs only on the host (ports bound to `127.0.0.1`). Containers on `ai-net` can access it via `http://minio:9000`, but external clients cannot reach it unless you change the host binding.
This document should help you and your colleagues understand the purpose of each section of the Compose file and how the services interoperate. Feel free to copy it into a README in your repository.
