# NextOps Ingestion Service - Azure VM Deployment Guide

A simple, step-by-step guide to deploy the **NextOps Telemetry Ingestion Service** on an Azure Virtual Machine (Ubuntu) with a public IP.

---

## 1. Prerequisites

- **Azure VM** running Ubuntu (22.04 / 24.04 LTS) with a **Public IP**.
- **SSH access** to the VM.
- **Port 8000 open** in the Azure Network Security Group (NSG).

---

## 2. Open Inbound Port in Azure (NSG)

To access the service via your Public IP:
1. Go to the [Azure Portal](https://portal.azure.com).
2. Navigate to your **Virtual Machine** > **Networking** (or **Network settings**).
3. Under **Inbound port rules**, click **Add inbound port rule**:
   - **Destination port ranges**: `8000`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Name**: `Allow-Port-8000`
4. Click **Add**.

---

## 3. Connect to Your VM

From your local terminal:
```bash
ssh <azure-username>@<YOUR_VM_PUBLIC_IP>
```

---

## 4. Install Docker

Docker is the simplest way to run both PostgreSQL and the ingestion service without dependency conflicts:

```bash
# Update packages and install Docker
sudo apt update
sudo apt install -y docker.io docker-compose

# Allow your user to run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker
```

---

## 5. Run PostgreSQL

Start a PostgreSQL container with a persistent volume:

```bash
docker run -d \
  --name nextops-postgres \
  --restart unless-stopped \
  -e POSTGRES_USER=axion_user \
  -e POSTGRES_PASSWORD=securepassword123 \
  -e POSTGRES_DB=axion_db \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

Initialize the database schema:
```bash
docker exec -i nextops-postgres psql -U axion_user -d axion_db << 'EOF'
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE IF NOT EXISTS telemetry (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id VARCHAR(50) NOT NULL,
    device_type VARCHAR(50) NOT NULL,
    refinery_region VARCHAR(50) NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    temperature DOUBLE PRECISION NOT NULL,
    pressure DOUBLE PRECISION NOT NULL,
    flow_rate DOUBLE PRECISION NOT NULL,
    component_metrics JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_telemetry_device_id_timestamp ON telemetry(device_id, timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_telemetry_timestamp ON telemetry(timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_telemetry_region ON telemetry(refinery_region);
EOF
```

---

## 6. Deploy NextOps Ingestion Service

### Clone & Configure

```bash
# Clone the repository (or copy the project directory)
git clone <YOUR_REPO_URL>
cd nextops-ingestion-service

# Create your .env file
cat << 'EOF' > .env
DATABASE_URL=postgresql://axion_user:securepassword123@172.17.0.1:5432/axion_db
EOF
```
> **Note**: `172.17.0.1` is the default Docker host bridge gateway IP, allowing the container to talk to PostgreSQL running on the VM host.

### Build and Run with Docker

```bash
# Build the Docker image
docker build -t nextops-ingestion-service .

# Run the container
docker run -d \
  --name nextops-ingestion \
  --restart unless-stopped \
  --env-file .env \
  -p 8000:8000 \
  nextops-ingestion-service
```

---

## 7. Verify the Deployment

Open your browser or run from your local machine:

1. **Health Check**:
   ```bash
   curl http://<YOUR_VM_PUBLIC_IP>:8000/health
   # Output: {"status":"UP"}
   ```

2. **Swagger UI (Interactive API Docs)**:
   ```
   http://<YOUR_VM_PUBLIC_IP>:8000/docs
   ```

3. **Send a Test Telemetry Record**:
   ```bash
   curl -X POST http://<YOUR_VM_PUBLIC_IP>:8000/api/v1/telemetry \
     -H "Content-Type: application/json" \
     -d '{
       "device_id": "BOILER-01",
       "device_type": "BOILER",
       "refinery_region": "STATION_1",
       "timestamp": "2026-11-14T12:00:00Z",
       "temperature": 540.5,
       "pressure": 165.2,
       "flow_rate": 2100.0,
       "component_metrics": {"steam_flow": 2100.0}
     }'
   ```

---

## Useful Maintenance Commands

```bash
# View service logs
docker logs -f nextops-ingestion

# Restart the service
docker restart nextops-ingestion

# Stop the service
docker stop nextops-ingestion
```
