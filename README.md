# Circuless Platform

Docker Compose–based deployment configuration for the Circuless data processing platform. The platform provides a modular, containerized environment for secure file exchange and data processing, built around an API gateway, flow engine, and S3-compatible storage.

- Reverse proxy and TLS termination via Traefik  
- Public and admin API gateway via Apache APISIX (backed by etcd)  
- Data processing flows (file upload/download pipelines) via Apache NiFi  
- Optional flow versioning via NiFi Registry  
- S3-compatible object storage via MinIO for file persistence  

All components are deployed as Docker services and interconnected via a shared external `traefik` Docker network.

## Overview
Circuless is designed as a lightweight, extensible data exchange platform focused on file-based workflows. Traefik terminates TLS and routes incoming traffic to APISIX, which serves as the central API gateway in front of NiFi-based processing pipelines. NiFi orchestrates upload and download flows, storing content in MinIO through an S3-compatible interface, while NiFi Registry can optionally be used to manage and version flow definitions across environments.


<!-- --- -->

<!-- ## Repository Structure

From `/opt`:

- `apisix/`
  - `docker-compose.yaml` – APISIX + etcd stack
  - `apisix-config/config.yaml` – APISIX configuration (routes, upstreams, plugins)
  - `etcd_data/` – persistent etcd data
- `nifi/`
  - `docker-compose.yaml` – NiFi + NiFi Registry + MinIO
  - `nifi/` – NiFi runtime data (repositories, logs, config, Python extensions)
  - `nifi_registry/` – NiFi Registry data
  - `minio/` – MinIO data (S3 object store)
- `traefik/`
  - `docker-compose.yml` – Traefik reverse proxy
  - `traefik.yml.example` – Traefik configuration template (copy to `traefik.yml` and adjust)
  - `letsencrypt/` – ACME certificate store (`acme.json`)
  - `create_network.sh`, `create_user.sh` – helper scripts (network/user setup)
- `LICENSE` – project license
- `README.md` – this documentation

All stacks share an external Docker network named `traefik`. -->

<!-- --- -->

## Services

### Core Platform Services

#### **Traefik** (Reverse Proxy & TLS Termination)

Entry point for all external HTTP(S) traffic. Handles TLS termination, ACME/Let’s Encrypt certificate management, and routing to backend services.

- **Technology**: Traefik v3.2
- **Compose file**: `traefik/docker-compose.yml`
- **Ports (host)**:
  - `80/tcp` → `web` entrypoint (HTTP, redirected to HTTPS)
  - `443/tcp` → `websecure` entrypoint (HTTPS)
- **Access**:
  - Fronts all public domains:
    - `APISIX_PUBLIC_DOMAIN`
    - `APISIX_ADMIN_DOMAIN`
    - `NIFI_DOMAIN`
    - `MINIO_DOMAIN`
- **Configuration**:
  - Static configuration via command-line arguments in `docker-compose.yml`
  - File-based configuration from `traefik.yml` (created from `traefik.yml.example`)
  - Certificates stored in `traefik/letsencrypt/acme.json`
- **Environment Variables** (in `traefik/.env`):
  - `ACME_EMAIL` – email address used for ACME/Let’s Encrypt registration
- **Persistent Data**:
  - `traefik/letsencrypt/` – ACME certificate storage
- **Dependencies**:
  - Docker socket (`/var/run/docker.sock`) for dynamic configuration from container labels
  - External Docker network `traefik`

---

### API Gateway & Configuration Store

#### **APISIX** (API Gateway)

Central API gateway for the platform. Handles routing, request processing, and integration with NiFi-based flows.

- **Technology**: Apache APISIX 3.13.0
- **Compose file**: `apisix/docker-compose.yaml`
- **Container name**: `circ-apisix`
- **Ports (internal)**:
  - `9080` – data plane (public API traffic)
  - `9180` – admin API (management/config)
- **Access**:
  - Public API via Traefik at `APISIX_PUBLIC_DOMAIN` (HTTPS)
  - Admin API via Traefik at `APISIX_ADMIN_DOMAIN` (HTTPS)
- **Configuration**:
  - Declarative configuration from `apisix/apisix-config/config.yaml`
  - Additional runtime configuration via Admin API (proxied through Traefik)
- **Environment Variables** (in `apisix/.env`):
  - `APISIX_PUBLIC_DOMAIN` – public API domain
  - `APISIX_ADMIN_DOMAIN` – admin API domain
- **Dependencies**:
  - `etcd` as configuration and service registry backend
  - `traefik` network for incoming traffic

#### **etcd** (Configuration Store for APISIX)

Holds APISIX runtime configuration and state.

- **Technology**: Bitnami etcd 3.6.4
- **Compose file**: `apisix/docker-compose.yaml`
- **Container name**: `circ-etcd`
- **Port (internal/exposed)**:
  - `2379` – client API (used by APISIX; optionally exposed on host)
- **Configuration**:
  - `ETCD_DATA_DIR=/bitnami/etcd`
  - `ETCD_ENABLE_V2=true`
  - `ALLOW_NONE_AUTHENTICATION=yes` (development/demo – review for production)
  - `ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379`
  - `ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379`
- **Persistent Data**:
  - `apisix/etcd_data/` → `/bitnami/etcd`
- **Dependencies**:
  - Used exclusively by APISIX in this platform

---

### Data Processing & Storage Stack

#### **Apache NiFi** (Data Flow Engine)

Implements data ingestion and processing flows (e.g. file upload/download). Receives traffic from APISIX and interacts with MinIO for object storage.

- **Technology**: Apache NiFi (official `apache/nifi:latest` image)
- **Compose file**: `nifi/docker-compose.yaml`
- **Container name**: `nifi`
- **Ports (internal)**:
  - HTTP: `8080` (configured via `NIFI_WEB_HTTP_PORT`)
  - HTTPS UI: `8443` (target of Traefik `nifi` service)
- **Access**:
  - Web UI via Traefik at `https://<NIFI_DOMAIN>` (HTTPS, port 8443 inside container)
- **Configuration**:
  - Environment variables from `nifi/.env`:
    - `NIFI_USERNAME` / `NIFI_PASSWORD` – single-user credentials
    - `NIFI_DOMAIN` – external domain used as `NIFI_WEB_PROXY_HOST`
  - NiFi repositories, state, and configuration mounted from:
    - `nifi/nifi/nifi-current/...` → `/opt/nifi/nifi-current/...`
  - Platform-provided flows stored as JSON definitions:
    - `nifi/Upload-flow.json`
    - `nifi/Download-flow.json`
- **Persistent Data**:
  - Repositories and state:
    - `database_repository`, `flowfile_repository`, `content_repository`,
      `provenance_repository`, `state`
  - Logs: `nifi/nifi/nifi-current/logs/`
  - Configuration: `nifi/nifi/nifi-current/conf/`
  - Extensions and libs: `nifi/nifi/nifi-current/python_extensions/`, `nifi/nifi/nifi-current/lib/`
- **Dependencies**:
  - Receives traffic from APISIX via Traefik
  - Uses MinIO as S3-compatible storage (for file handling flows)

#### **NiFi Registry** 



- **Technology**: Apache NiFi Registry 2.3.0
- **Compose file**: `nifi/docker-compose.yaml`
- **Container name**: `registry_container_persistent`
- **Ports**:
  - Internal default (not exposed via Traefik by default)
- **Configuration**:
  - File-based flow storage:
    - `NIFI_REGISTRY_DB_DIR=/opt/nifi-registry/nifi-registry-current/database`
    - `NIFI_REGISTRY_FLOW_PROVIDER=file`
    - `NIFI_REGISTRY_FLOW_STORAGE_DIR=/opt/nifi-registry/nifi-registry-current/flow_storage`
- **Persistent Data**:
  - `nifi/nifi_registry/database/`
  - `nifi/nifi_registry/flow_storage/`
- **Dependencies**:
  - NiFi can connect to Registry for flow versioning (optional; not configured by default)

#### **MinIO** (S3-Compatible Object Storage)

Stores files and objects for NiFi flows (e.g. uploaded/downloaded content). Provides an S3-compatible API and web console.

- **Technology**: MinIO (image `minio/minio:RELEASE.2025-03-12T18-04-18Z`)
- **Compose file**: `nifi/docker-compose.yaml`
- **Container name**: `minio`
- **Ports (internal)**:
  - `9001` – console (configured via `--console-address ":9001"`)
  - S3 API: default MinIO port (internal, used by NiFi; not exposed directly via Traefik)
- **Access**:
  - Console via Traefik at `https://<MINIO_DOMAIN>` (mapped to port 9001 inside container)
- **Configuration**:
  - Environment variables from `nifi/.env`:
    - `MINIO_ACCESS_KEY` → `MINIO_ROOT_USER`
    - `MINIO_SECRET_KEY` → `MINIO_ROOT_PASSWORD`
- **Persistent Data**:
  - `nifi/minio/` → `/data` (buckets and stored objects)
- **Healthcheck**:
  - `http://127.0.0.1:9001/minio/health/live`
- **Dependencies**:
  - Used by NiFi as S3 storage backend for file upload/download flows


---

## Deployment

### 1. Clone Repository

    cd /opt
    git clone <repository-url> circuless-platform
    cd circuless-platform

(Adjust the repository URL and directory name as needed.)

### 2. Create the External `traefik` Network

If not already created:

    cd traefik
    ./create_network.sh



Ensure all stacks (`traefik`, `apisix`, `nifi`) refer to `networks: traefik: external: true`.

### 3. Configure Environment Files

For each module, create `.env` files next to the respective `docker-compose` files using the provided examples.

#### Traefik (`traefik/.env`)

    ACME_EMAIL=your-email@example.com

#### APISIX (`apisix/.env`)

    APISIX_PUBLIC_DOMAIN=your-domain.example.com
    APISIX_ADMIN_DOMAIN=apisix-admin.your-domain.example.com

#### NiFi Stack (`nifi/.env`)

    NIFI_USERNAME=admin
    NIFI_PASSWORD=change-me-to-strong-password
    NIFI_DOMAIN=nifi.your-domain.example.com

    MINIO_ACCESS_KEY=change-me-to-strong-access-key
    MINIO_SECRET_KEY=change-me-to-strong-secret
    MINIO_DOMAIN=s3.your-domain.example.com


### 4. Start Traefik

    cd traefik
    docker compose up -d

This will:

- Start Traefik
- Bind ports `80` and `443`
- Initialize ACME storage (`letsencrypt/acme.json`)

### 5. Start APISIX + etcd

    cd ../apisix
    docker compose up -d

This will:

- Start etcd and APISIX
- Expose:
  - APISIX public APIs via `APISIX_PUBLIC_DOMAIN` through Traefik
  - APISIX Admin API via `APISIX_ADMIN_DOMAIN` through Traefik

Ensure `apisix-config/config.yaml` contains appropriate routes, upstreams, and plugins for your use case.

### 6. First-Time NiFi Configuration (one-time setup)
For the first deployment only, NiFi’s `conf` directory must be initialized from a clean upstream image.

#### Prepare directory structure:

```
  cd ../nifi
  mkdir -p nifi/nifi-current
```
  

#### Start a temporary NiFi container:
  
  ```
   docker run --rm -it --entrypoint bash apache/nifi:latest 
  ```

#### Copy the `conf` directory from the temporary container:

```
docker cp <container-id>:/opt/nifi/nifi-current/conf ./nifi/nifi-current/conf
```

#### Stop/exit the temporary container:

#### Edit NiFi properties:
```
nano ./nifi/nifi-current/conf/nifi.properties
```

##### Adjust:

```
nifi.web.proxy.host=your.domain.com
nifi.web.https.host=change_me
```

`nifi.web.proxy.host` must match NIFI_DOMAIN (the external hostname used via Traefik).

`nifi.web.https.host` should be set appropriately for your deployment (host/IP NiFi binds to for HTTPS).

### 7. Start NiFi, NiFi Registry, and MinIO

    cd ../nifi
    docker compose up -d

This will:

- Start NiFi (accessible via `https://NIFI_DOMAIN`)
- Start NiFi Registry
- Start MinIO (console accessible via `https://MINIO_DOMAIN`)

### 8. Traefik and NiFi 

Add  this  to `traefik.yml` 

```
serversTransport:
  insecureSkipVerify: true
```


## Service Architecture

### High-Level Architecture

    Internet
      ↓
    Traefik (TLS termination, routing)
      ↓
      ├─→ APISIX Public API (port 9080)
      │      └─→ NiFi (data flows)
      │             └─→ MinIO (S3 object storage)
      └─→ APISIX Admin API (port 9180)

- All service-to-service communication happens on the internal `traefik` Docker network.
- Traefik handles HTTPS for all externally exposed endpoints.
- APISIX acts as API gateway in front of NiFi-based integrations and other backend services.

### APISIX Configuration Flow

Configuration of APISIX is typically done via the Admin API:

    Admin
       ↓
    Internet
       ↓
    Traefik (websecure, APISIX_ADMIN_DOMAIN)
       ↓
    APISIX Admin API (9180)
       ↓
    etcd (configuration storage)
       ↓
    APISIX Data Plane (9080 – executes routes/plugins)



### File Upload Flow

    Client (Uploader)
       ↓
    Internet
       ↓
    Traefik (websecure, APISIX_PUBLIC_DOMAIN)
       ↓
    APISIX Public API (9080 – routing, auth, rate limiting, etc.)
       ↓
    NiFi (ingestion & processing flow)
       ↓
    MinIO (S3 bucket – persistent file storage)

Typical responsibilities:

- APISIX:
  - Authentication/authorization
  - Rate limiting, logging, additional plugins
  - Routing to NiFi HTTP endpoints
- NiFi:
  - Receive files (HTTP/REST)
  - Perform transformations, validation, enrichment
  - Write files/objects to MinIO (S3 API)
- MinIO:
  - Store and serve binary objects

### File Download Flow

    Client (Downloader)
       ↓
    Internet
       ↓
    Traefik (websecure, APISIX_PUBLIC_DOMAIN)
       ↓
    APISIX Public API
       ↓
    NiFi (download flow – read from S3, apply business rules)
       ↓
    MinIO (S3 object storage)

Depending on configuration, NiFi may either:

- Stream content directly from MinIO to the client via APISIX, or
- Pre-process/prepare data from MinIO and return it to the client.

---

## Maintenance

### View Logs

From the respective directory:

- All services in a stack:

      docker compose logs -f

- Specific service (example: NiFi):

      docker compose logs -f nifi

- Last 100 lines:

      docker compose logs --tail=100 nifi


### Update Services

To pull newer images and restart:

    # Traefik
    cd traefik
    docker compose pull
    docker compose up -d

    # APISIX
    cd ../apisix
    docker compose pull
    docker compose up -d

    # NiFi stack
    cd ../nifi
    docker compose pull
    docker compose up -d


## License

This project is licensed under the terms specified in the `LICENSE` file in the repository root.

---


## Support

For issues specific to:
- **Circuless services**: Contact bAvenir support at [support@bavenir.com](mailto:support@bavenir.com)
- **Docker/Infrastructure**: Check Docker, Traefik and Aisix documentation
