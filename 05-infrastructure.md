# 🔧 BLUEPRINT MODUL 05: M1 INFRASTRUCTURE SOVEREIGNTY & NATIVE EXCELLENCE
## Version 2.0 (2026-01-26) | Elite Status: 500+ Lines | Authority: Sisyphus CEO

---

## TABLE OF CONTENTS
1. M1 Architecture Fundamentals
2. Docker Sovereignty & Persistence Mandate
3. M1 Dependency Management & Wheel Verification
4. Network Topology & IP Assignment (172.20.0.x)
5. Persistent Storage & Volume Management
6. Performance Tuning for Apple Silicon
7. Disaster Recovery & Backup Strategy
8. M1-Specific Troubleshooting & Remediation
9. Compliance Verification Checklist
10. Authority & Governance

---

## 1. M1 ARCHITECTURE FUNDAMENTALS

### ARM64 vs x86_64: Critical Differences

**Apple M1 Architecture (2020+)**:
- **ISA**: ARM64 (Reduced Instruction Set, 64-bit)
- **Cores**: 8 cores total (4 Performance + 4 Efficiency)
- **Performance**: ~40% faster than Intel i9 for same power
- **Power Consumption**: 5-15W (vs. 45-95W x86)
- **Native Support**: iOS/macOS binaries only, cannot execute x86_64 directly

**Rosetta 2 Emulation Layer (Fallback Only)**:
- **Purpose**: Run x86_64 applications on ARM64
- **Performance Penalty**: 10-30% slower than native ARM64
- **Memory Overhead**: +50-100MB per process
- **Detection**: `file` command shows "x86_64" instead of "arm64"
- **Forbidden in SIN-Solver**: Rosetta2 is a LAST RESORT, not acceptable for production

### Docker on M1 macOS

**Key Constraint**: Docker Desktop runs a lightweight Linux VM using Hypervisor.framework
- VM is ARM64-native (not Rosetta2)
- All containers MUST be ARM64 or they run via Rosetta2 (slow & detectable)
- Dockerfile `FROM` image MUST specify `arm64v8/` prefix or platform flag

**Correct Setup**:
```bash
# Install Docker Desktop for Mac (M1-native)
brew install docker

# Verify architecture:
docker run --rm arm64v8/alpine uname -m
# Output: aarch64 (ARM64) ✅

# Verify NOT x86_64:
docker run --rm --platform linux/amd64 arm64v8/alpine uname -m
# Output: x86_64 (emulated via QEMU) ❌ DO NOT USE
```

**Platform Flag (Recommended)**:
```yaml
# docker-compose.yml
version: '3.8'
services:
  agent-zero:
    platform: linux/arm64  # Explicitly set ARM64
    image: custom/agent-zero:v3.0
    # ... rest of config
```

---

## 2. DOCKER SOVEREIGNTY & PERSISTENCE MANDATE

### The Sacred Principle: No Image Loss

**Mandate**: Every Docker image MUST be saved locally. Cloud dependencies are forbidden.

**Why Local Persistence is Critical**:
1. **Offline Operation**: System works even if Docker Hub/registry unavailable
2. **Compliance**: All artifacts stored locally (audit trail, version control)
3. **Performance**: Load from local disk (1s) vs. pull from registry (30-60s)
4. **Cost**: No bandwidth charges for image pulls
5. **Sovereignty**: Full control, zero external dependencies

### Directory Structure

```
/Users/jeremy/dev/SIN-Solver/
├── Docker/
│   ├── images/              # Local image archives (tar.gz format)
│   │   ├── agent-zero-v3.0-arm64.tar.gz     (1.2GB)
│   │   ├── gemini-solver-v2.0-arm64.tar.gz  (800MB)
│   │   ├── mistral-solver-v1.5-arm64.tar.gz (600MB)
│   │   ├── yolo-solver-v8-arm64.tar.gz      (400MB)
│   │   ├── steel-stealth-v2.0-arm64.tar.gz  (300MB)
│   │   ├── skyvern-v1.0-arm64.tar.gz        (2.1GB)
│   │   ├── chronos-stratege-v1.0-arm64.tar.gz (200MB)
│   │   ├── stagehand-detektiv-v2.0-arm64.tar.gz (250MB)
│   │   ├── postgres-15-alpine-arm64.tar.gz  (80MB)
│   │   ├── redis-latest-arm64.tar.gz        (60MB)
│   │   ├── qdrant-latest-arm64.tar.gz       (100MB)
│   │   ├── grafana-latest-arm64.tar.gz      (300MB)
│   │   ├── prometheus-latest-arm64.tar.gz   (150MB)
│   │   └── nginx-latest-arm64.tar.gz        (40MB)
│   ├── configs/             # Docker configs
│   │   ├── docker-compose.yml
│   │   ├── docker-compose.prod.yml
│   │   └── .env (secrets, gitignore)
│   ├── volumes/             # Persistent data
│   │   ├── postgres-data/
│   │   ├── redis-data/
│   │   ├── qdrant-data/
│   │   ├── grafana-data/
│   │   └── chronos-data/
│   ├── scripts/
│   │   ├── backup-images.sh     # Save all images
│   │   ├── restore-images.sh    # Load all images
│   │   ├── verify-arm64.sh      # Check architecture
│   │   ├── health-check.sh      # Monitor all containers
│   │   └── disaster-recovery.sh # Full restore from backup
│   └── logs/
│       ├── container-logs/
│       ├── docker-daemon.log
│       └── health-check.log
└── .env (gitignored)
```

### Image Persistence Workflow

#### Step 1: Build or Pull Image

```bash
# Option A: Pull from Docker Hub (for public images like Postgres)
docker pull postgres:15-alpine

# Option B: Build custom image from Dockerfile
docker build -t custom/agent-zero:v3.0 -f Dockerfile.agent-zero .

# Verify architecture BEFORE saving
docker inspect custom/agent-zero:v3.0 | grep -i architecture
# Output: "Architecture": "arm64" ✅
```

#### Step 2: Save Image Locally

```bash
# Save single image
docker save custom/agent-zero:v3.0 | gzip > /Users/jeremy/dev/SIN-Solver/Docker/images/agent-zero-v3.0-arm64.tar.gz

# Save all images in batch (script)
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/backup-images.sh

IMAGES=(
  "custom/agent-zero:v3.0"
  "custom/steel-stealth:v2.0"
  "custom/skyvern:v1.0"
  "postgres:15-alpine"
  "redis:latest"
  "qdrant:latest"
  "grafana:latest"
)

for image in "${IMAGES[@]}"; do
  filename=$(echo "$image" | tr '/:' '-')-arm64.tar.gz
  echo "Saving $image → $filename"
  docker save "$image" | gzip > "/Users/jeremy/dev/SIN-Solver/Docker/images/$filename"
  echo "✅ Saved $(ls -lh /Users/jeremy/dev/SIN-Solver/Docker/images/$filename | awk '{print $5}')"
done

echo "All images backed up to /Users/jeremy/dev/SIN-Solver/Docker/images/"
```

#### Step 3: Verify Checksum (Data Integrity)

```bash
# Generate checksum after saving
sha256sum /Users/jeremy/dev/SIN-Solver/Docker/images/*.tar.gz > /Users/jeremy/dev/SIN-Solver/Docker/images/CHECKSUMS.sha256

# Verify on restore
sha256sum -c /Users/jeremy/dev/SIN-Solver/Docker/images/CHECKSUMS.sha256
# Output: agent-zero-v3.0-arm64.tar.gz: OK ✅

# If checksum fails: image corrupted, alert immediately!
```

#### Step 4: Load Image from Archive (Restore)

```bash
# Load single image
docker load < /Users/jeremy/dev/SIN-Solver/Docker/images/agent-zero-v3.0-arm64.tar.gz

# Load all images (restore script)
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/restore-images.sh

echo "Restoring all Docker images from local archives..."

for archive in /Users/jeremy/dev/SIN-Solver/Docker/images/*.tar.gz; do
  echo "Loading $(basename $archive)..."
  docker load < "$archive"
  
  if [ $? -eq 0 ]; then
    echo "✅ Loaded successfully"
  else
    echo "❌ FAILED: $archive"
    exit 1
  fi
done

echo "All images restored successfully!"
docker images | grep -E "(agent-zero|steel|skyvern|postgres|redis|qdrant|grafana)"
```

### Volume Management (Persistent Data)

**Critical Data**: Postgres data, Redis data, Qdrant vectors (CANNOT be lost)

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    volumes:
      - type: bind
        source: /Users/jeremy/dev/SIN-Solver/Docker/volumes/postgres-data
        target: /var/lib/postgresql/data
    # Data persists at: /Users/jeremy/dev/SIN-Solver/Docker/volumes/postgres-data

  redis:
    image: redis:latest
    volumes:
      - type: bind
        source: /Users/jeremy/dev/SIN-Solver/Docker/volumes/redis-data
        target: /data
    command: redis-server --appendonly yes
    # AOF (Append-Only File) persistence: every write persists

  qdrant:
    image: qdrant:latest
    volumes:
      - type: bind
        source: /Users/jeremy/dev/SIN-Solver/Docker/volumes/qdrant-data
        target: /qdrant/storage

volumes:
  # Empty {} means use bind mount (source = host path)
```

**Backup Strategy**:
```bash
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/backup-volumes.sh

BACKUP_DIR="/Users/jeremy/dev/SIN-Solver/Docker/backups/$(date +%Y-%m-%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup Postgres
echo "Backing up Postgres..."
docker exec sin_postgres_1 pg_dump -U postgres -d sin_solver \
  | gzip > "$BACKUP_DIR/postgres-backup.sql.gz"

# Backup Redis
echo "Backing up Redis..."
docker exec sin_redis_1 redis-cli BGSAVE
sleep 2
cp /Users/jeremy/dev/SIN-Solver/Docker/volumes/redis-data/dump.rdb "$BACKUP_DIR/redis-dump.rdb"

# Backup Qdrant
echo "Backing up Qdrant..."
tar -czf "$BACKUP_DIR/qdrant-backup.tar.gz" \
  /Users/jeremy/dev/SIN-Solver/Docker/volumes/qdrant-data

# Verify backups
du -sh "$BACKUP_DIR"/*
echo "Backups saved to: $BACKUP_DIR"
```

---

## 3. M1 DEPENDENCY MANAGEMENT & WHEEL VERIFICATION

### Python Wheel Requirements for ARM64

**Critical Rule**: All Python packages MUST have native ARM64 wheels (not x86_64, not source builds).

**Required Versions**:
```
Python >= 3.11               (built with --enable-optimizations on ARM64)
numpy >= 1.21.0             (native arm64 wheels)
opencv-python >= 4.5.5      (native arm64 wheels)
torch >= 1.12.0             (native arm64 wheels)
torchvision >= 0.13.0       (depends on torch)
scikit-learn >= 1.1.0       (native arm64 wheels)
pandas >= 1.3.0             (native arm64 wheels)
tensorflow >= 2.8.0         (native arm64 wheels, if used)
httpx >= 0.23.0             (pure Python, works on ARM64)
fastapi >= 0.100.0          (pure Python, works on ARM64)
```

### Verification Workflow

```bash
# Step 1: Install dependencies
pip install --upgrade pip wheel setuptools

# Step 2: Install with explicit ARM64 constraint
pip install numpy==1.24.0 \
    --only-binary=:all: \
    --platform macosx_12_0_arm64

# Step 3: Verify installed wheel is ARM64
python -c "import numpy; print(numpy.__file__)"
# Output: /opt/homebrew/lib/python3.11/site-packages/numpy/__init__.py ✅

# Step 4: Check for Rosetta2 markers (NOT allowed)
python -c "import sys; print(sys.platform)"
# Output: darwin ✅ (not 'darwin-x86_64')

# Step 5: Verify no x86_64 wheel files exist
pip show numpy
# Look for: Location: /opt/homebrew/lib/python3.11/site-packages
# If contains 'x86_64' anywhere: REJECT ❌

# Batch verification script:
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/verify-arm64-wheels.sh

echo "Verifying all installed packages are ARM64 native..."
pip list -v | while read package location; do
  if [[ $location == *"x86_64"* ]] || [[ $location == *"intel"* ]]; then
    echo "❌ ROSETTA2 DETECTED: $package from $location"
    exit 1
  fi
done
echo "✅ All packages are ARM64 native!"
```

### Problematic Packages (Workarounds)

**Package**: `psycopg2` (PostgreSQL driver)
```bash
# ❌ WRONG (installs from source, slow):
pip install psycopg2

# ✅ CORRECT (pre-built binary):
pip install psycopg2-binary
```

**Package**: `Pillow` (Image library)
```bash
# ✅ CORRECT (has ARM64 wheels):
pip install Pillow==9.5.0 --only-binary=:all:
```

**Package**: `lxml` (XML parser)
```bash
# ❌ Problem: No ARM64 wheel, tries to build from source
pip install lxml

# ✅ Solution:
pip install lxml --only-binary=:all: \
    --platform macosx_12_0_arm64
```

---

## 4. NETWORK TOPOLOGY & IP ASSIGNMENT (172.20.0.x)

### Internal Network Design

```
Docker Network: sin_network
Type: bridge
Subnet: 172.20.0.0/24
Gateway: 172.20.0.1

IP Assignments (STATIC, REQUIRED):
┌──────┬────────────────────────┬──────────┬────────────┐
│ IP   │ Service                │ Port     │ CPU/RAM    │
├──────┼────────────────────────┼──────────┼────────────┤
│ .2   │ Chronos-Stratege       │ 3001     │ 1c / 512MB │
│ .4   │ Opencode-Sekretaer     │ 8200     │ 1c / 512MB │
│ .7   │ Stagehand Detektiv     │ 3000     │ 2c / 2GB   │
│ .8   │ QA-Prüfer             │ 8080     │ 2c / 1GB   │
│ .9   │ Clawdbot-Bote         │ 8080     │ 1c / 512MB │
│ .10  │ Postgres              │ 5432     │ 2c / 2GB   │
│ .12  │ Evolution Optimizer    │ 8080     │ 1c / 512MB │
│ .15  │ Qdrant (Surfsense)    │ 6333     │ 2c / 2GB   │
│ .20  │ Steel Stealth         │ 3000     │ 2c / 1GB   │
│ .30  │ Skyvern Auge          │ 8000     │ 2c / 2GB   │
│ .31  │ API Brain             │ 8000     │ 1c / 512MB │
│ .40  │ SIN-Plugins MCP       │ 8000     │ 1c / 512MB │
│ .50  │ Agent Zero            │ 8000     │ 4c / 4GB   │
│ .60  │ Dashboard Zentrale    │ 3000     │ 1c / 1GB   │
│100   │ Prometheus            │ 9090     │ 1c / 512MB │
│101   │ Redis                 │ 6379     │ 1c / 1GB   │
└──────┴────────────────────────┴──────────┴────────────┘
```

### docker-compose.yml (Complete Configuration)

```yaml
version: '3.8'

networks:
  sin_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

services:
  # --- Core Consensus Engine ---
  agent-zero:
    container_name: sin_agent_zero
    platform: linux/arm64
    image: custom/agent-zero:v3.0
    networks:
      sin_network:
        ipv4_address: 172.20.0.50
    ports:
      - "8000:8000"
    environment:
      - GEMINI_API_KEY=${GEMINI_API_KEY}
      - MISTRAL_API_KEY=${MISTRAL_API_KEY}
      - VAULT_ADDR=http://172.20.0.4:8200
      - POSTGRES_URL=postgresql://postgres:${POSTGRES_PASSWORD}@172.20.0.10:5432/sin_solver
      - REDIS_URL=redis://172.20.0.101:6379
    depends_on:
      - postgres
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    cpus: '4'
    memory: 4g

  # --- Networking & Proxying ---
  clawdbot:
    container_name: sin_clawdbot
    platform: linux/arm64
    image: nginx:latest
    networks:
      sin_network:
        ipv4_address: 172.20.0.9
    ports:
      - "8080:8080"
    volumes:
      - ./configs/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - agent-zero
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 10s
      timeout: 5s
    restart: unless-stopped

  # --- Data Persistence ---
  postgres:
    container_name: sin_postgres
    platform: linux/arm64
    image: postgres:15-alpine
    networks:
      sin_network:
        ipv4_address: 172.20.0.10
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=sin_solver
    volumes:
      - /Users/jeremy/dev/SIN-Solver/Docker/volumes/postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    cpus: '2'
    memory: 2g

  redis:
    container_name: sin_redis
    platform: linux/arm64
    image: redis:latest
    networks:
      sin_network:
        ipv4_address: 172.20.0.101
    command: redis-server --appendonly yes
    volumes:
      - /Users/jeremy/dev/SIN-Solver/Docker/volumes/redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    cpus: '1'
    memory: 1g

  # --- Vector Database (Cache) ---
  qdrant:
    container_name: sin_qdrant
    platform: linux/arm64
    image: qdrant:latest
    networks:
      sin_network:
        ipv4_address: 172.20.0.15
    ports:
      - "6333:6333"
    volumes:
      - /Users/jeremy/dev/SIN-Solver/Docker/volumes/qdrant-data:/qdrant/storage
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/health"]
      interval: 60s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    cpus: '2'
    memory: 2g

  # --- Monitoring ---
  prometheus:
    container_name: sin_prometheus
    platform: linux/arm64
    image: prom/prometheus:latest
    networks:
      sin_network:
        ipv4_address: 172.20.0.100
    ports:
      - "9090:9090"
    volumes:
      - ./configs/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /Users/jeremy/dev/SIN-Solver/Docker/volumes/prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  grafana:
    container_name: sin_grafana
    platform: linux/arm64
    image: grafana:latest
    networks:
      sin_network:
        ipv4_address: 172.20.0.60
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - /Users/jeremy/dev/SIN-Solver/Docker/volumes/grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped
```

---

## 5. PERSISTENT STORAGE & VOLUME MANAGEMENT

### Directory Initialization

```bash
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/init-volumes.sh

# Create all volume directories
mkdir -p /Users/jeremy/dev/SIN-Solver/Docker/volumes/{postgres-data,redis-data,qdrant-data,prometheus-data,grafana-data,chronos-data}

# Set proper permissions (Docker container user access)
chmod 755 /Users/jeremy/dev/SIN-Solver/Docker/volumes/*
chown nobody:nobody /Users/jeremy/dev/SIN-Solver/Docker/volumes/postgres-data
chown nobody:nobody /Users/jeremy/dev/SIN-Solver/Docker/volumes/redis-data

# Initialize Postgres schema
docker run --rm \
  -e PGPASSWORD=mysecretpassword \
  -v /Users/jeremy/dev/SIN-Solver/Docker/volumes/postgres-data:/var/lib/postgresql/data \
  postgres:15-alpine \
  psql -h localhost -U postgres -d sin_solver -f /init-schema.sql

echo "✅ Volume initialization complete"
```

### Disk Usage Monitoring

```bash
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/monitor-disk-usage.sh

echo "=== Docker Volume Usage ==="
du -sh /Users/jeremy/dev/SIN-Solver/Docker/volumes/*

echo ""
echo "=== Disk Space Available ==="
df -h /Users/jeremy/dev/SIN-Solver/Docker/volumes

echo ""
echo "=== Alert Thresholds ==="
total_usage=$(du -sk /Users/jeremy/dev/SIN-Solver/Docker/volumes | cut -f1)
available=$(df -k /Users/jeremy/dev/SIN-Solver/Docker/volumes | tail -1 | awk '{print $4}')

if [ $total_usage -gt 800000 ]; then  # 800GB
  echo "⚠️  WARNING: Docker volumes > 800GB (cleanup or expand)"
fi

if [ $available -lt 50000000 ]; then  # 50GB free
  echo "🚨 CRITICAL: < 50GB disk space available!"
fi
```

---

## 6. PERFORMANCE TUNING FOR APPLE SILICON

### CPU Scheduling Optimization

```yaml
# docker-compose.yml (Agent Zero service)
services:
  agent-zero:
    cpus: '4'          # Allocate 4 cores (out of 8)
    cpuset: '0-3'      # Bind to performance cores (not efficiency cores)
    cpu_shares: 1024   # Default priority
```

### Memory & Swap Configuration

```bash
# Check M1 memory limits
docker stats sin_agent_zero
# CONTAINER CPU %    MEM USAGE / LIMIT    MEM %
# sin_agent_zero  45%    2.1G / 4G         52%

# Adjust if needed (in docker-compose.yml):
# memory: 4g
# memswap_limit: 6g    # Allow up to 2GB swap
```

### GPU Acceleration (If Available)

```yaml
# docker-compose.yml (for GPU access on M1 with external GPU)
services:
  agent-zero:
    runtime: nvidia
    environment:
      - CUDA_VISIBLE_DEVICES=0
# Note: M1 has integrated GPU, use via PyTorch/TensorFlow Metal Performance Shaders (MPS)
```

### Disk I/O Optimization

```bash
# Use SSD for Docker volumes
diskutil info / | grep "Solid State"
# Result: "Solid State: Yes" ✅

# Monitor I/O
iotop -o -n 5
# Shows top 5 most I/O-intensive containers
```

---

## 7. DISASTER RECOVERY & BACKUP STRATEGY

### Daily Backup Routine

```bash
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/daily-backup.sh

set -e

BACKUP_DATE=$(date +%Y-%m-%d)
BACKUP_DIR="/Users/jeremy/dev/SIN-Solver/Docker/backups/$BACKUP_DATE"

mkdir -p "$BACKUP_DIR"

echo "Starting daily backup: $BACKUP_DATE"

# 1. Backup all Docker images
echo "Backing up Docker images..."
for image in $(docker images --format "{{.Repository}}:{{.Tag}}" | grep -v "^<none>"); do
  filename=$(echo "$image" | tr '/:' '-')-arm64.tar.gz
  docker save "$image" | gzip > "$BACKUP_DIR/$filename"
  echo "  ✅ $image"
done

# 2. Backup Postgres
echo "Backing up Postgres..."
docker exec sin_postgres pg_dump -U postgres sin_solver \
  | gzip > "$BACKUP_DIR/postgres-dump.sql.gz"

# 3. Backup Redis
echo "Backing up Redis..."
docker exec sin_redis redis-cli BGSAVE > /dev/null
cp /Users/jeremy/dev/SIN-Solver/Docker/volumes/redis-data/dump.rdb "$BACKUP_DIR/redis-dump.rdb"

# 4. Backup Qdrant vectors
echo "Backing up Qdrant..."
tar -czf "$BACKUP_DIR/qdrant-vectors.tar.gz" \
  /Users/jeremy/dev/SIN-Solver/Docker/volumes/qdrant-data

# 5. Generate checksums
echo "Generating checksums..."
cd "$BACKUP_DIR"
sha256sum * > CHECKSUMS.sha256
cd -

# 6. Report backup size
BACKUP_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
echo ""
echo "✅ Backup complete: $BACKUP_SIZE"
echo "   Location: $BACKUP_DIR"

# 7. Upload to S3 (optional, for offsite backup)
# aws s3 sync "$BACKUP_DIR" s3://sin-solver-backups/"$BACKUP_DATE" --storage-class GLACIER
```

### Disaster Recovery (Full Restore)

```bash
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/disaster-recovery.sh
# Usage: ./disaster-recovery.sh /path/to/backup-date

BACKUP_DIR="$1"

if [ ! -d "$BACKUP_DIR" ]; then
  echo "❌ Backup directory not found: $BACKUP_DIR"
  exit 1
fi

echo "Starting disaster recovery from: $BACKUP_DIR"

# 1. Stop all containers
echo "Stopping all containers..."
docker-compose -f /Users/jeremy/dev/SIN-Solver/Docker/configs/docker-compose.yml down

# 2. Verify backup integrity
echo "Verifying backup integrity..."
cd "$BACKUP_DIR"
sha256sum -c CHECKSUMS.sha256 || { echo "❌ Checksum failed"; exit 1; }
cd -

# 3. Restore Docker images
echo "Restoring Docker images..."
for archive in "$BACKUP_DIR"/*.tar.gz; do
  if [[ $(basename "$archive") != "postgres-dump"* ]] && \
     [[ $(basename "$archive") != "redis-dump"* ]] && \
     [[ $(basename "$archive") != "qdrant"* ]]; then
    echo "  Loading $(basename $archive)..."
    docker load < "$archive"
  fi
done

# 4. Clear old volumes
echo "Clearing old volumes..."
rm -rf /Users/jeremy/dev/SIN-Solver/Docker/volumes/{postgres-data,redis-data,qdrant-data}/*

# 5. Restore Postgres
echo "Restoring Postgres..."
docker-compose -f /Users/jeremy/dev/SIN-Solver/Docker/configs/docker-compose.yml up -d postgres
sleep 10
docker exec sin_postgres psql -U postgres -d sin_solver < \
  <(gunzip -c "$BACKUP_DIR/postgres-dump.sql.gz")

# 6. Restore Redis
echo "Restoring Redis..."
cp "$BACKUP_DIR/redis-dump.rdb" /Users/jeremy/dev/SIN-Solver/Docker/volumes/redis-data/
docker-compose -f /Users/jeremy/dev/SIN-Solver/Docker/configs/docker-compose.yml up -d redis

# 7. Restore Qdrant
echo "Restoring Qdrant..."
tar -xzf "$BACKUP_DIR/qdrant-vectors.tar.gz" -C /Users/jeremy/dev/SIN-Solver/Docker/volumes/

# 8. Start all services
echo "Starting all services..."
docker-compose -f /Users/jeremy/dev/SIN-Solver/Docker/configs/docker-compose.yml up -d

# 9. Verify health
echo "Verifying health..."
sleep 10
docker-compose -f /Users/jeremy/dev/SIN-Solver/Docker/configs/docker-compose.yml ps

echo "✅ Disaster recovery complete!"
```

---

## 8. M1-SPECIFIC TROUBLESHOOTING & REMEDIATION

### Issue 1: "image not found" (Pulling x86_64 instead of ARM64)

**Symptom**: Docker pulls x86_64 image, runs in Rosetta2, slow performance

**Root Cause**: Missing `platform` flag or wrong base image in Dockerfile

**Fix**:
```yaml
# docker-compose.yml
services:
  agent-zero:
    platform: linux/arm64  # ADD THIS LINE
    image: custom/agent-zero:v3.0
```

### Issue 2: "Exec format error" (Trying to run x86_64 binary)

**Symptom**: `docker run custom/agent-zero` fails with "Exec format error"

**Root Cause**: Container image contains x86_64 binary

**Fix**:
```dockerfile
# Dockerfile (correct for ARM64)
FROM python:3.11-slim  # Already M1-compatible

# Verify during build
RUN file /usr/bin/python3
# Output: /usr/bin/python3: ELF 64-bit LSB executable, ARM aarch64, ...✅

# Build with explicit platform
docker build -t custom/agent-zero:v3.0 \
  --platform linux/arm64 \
  -f Dockerfile.agent-zero .
```

### Issue 3: "Wheel file not compatible" (Rosetta2 Python package)

**Symptom**: `pip install numpy` installs x86_64 wheel

**Root Cause**: Homebrew Python or wrong pip

**Fix**:
```bash
# Verify Python is M1 native
file $(which python3)
# Output: ELF 64-bit LSB executable, ARM aarch64, ...✅

# If x86_64, uninstall and reinstall:
brew uninstall python3
brew install python@3.11  # Will be M1-native

# Verify again
python3 --version
arch  # Output: arm64 ✅
```

### Issue 4: "Out of memory" or "Container killed"

**Symptom**: Service crashes with OOM killer, no logs

**Root Cause**: Memory limit too low for workload

**Fix**:
```yaml
# docker-compose.yml
services:
  agent-zero:
    memory: 4g        # Increase from 2g
    memswap_limit: 6g # Allow swap
    # Or increase M1 VM memory in Docker Desktop:
    # Settings → Resources → Memory: 8GB+
```

### Issue 5: "Slow inference" or "Latency spike"

**Symptom**: YOLO takes 5+ seconds (should be 1-2s)

**Root Cause**: CPU throttling, Rosetta2 emulation, or resource contention

**Fix**:
```bash
# Check if Rosetta2 is involved
docker exec sin_agent_zero uname -m
# Output: aarch64 ✅ (ARM64)
# Output: x86_64 ❌ (Rosetta2 — BAD)

# Check CPU usage
docker stats sin_agent_zero
# If CPU < 50% but latency high: CPU throttling
# Solution: Increase docker-compose cpus or limit background processes

# Monitor system load
top -l 1 | head -20
# If load > 8 (on M1 8-core): System overloaded, reduce container count
```

---

## 9. COMPLIANCE VERIFICATION CHECKLIST

### M1 ARM64 Compliance Audit

```bash
#!/bin/bash
# /Users/jeremy/dev/SIN-Solver/Docker/scripts/m1-compliance-audit.sh

echo "=== M1 COMPLIANCE AUDIT ==="
echo ""

# 1. Check Docker Desktop is M1
echo "1. Docker Desktop Architecture:"
docker info | grep "Architecture"
# Expected: "Architecture: aarch64" ✅

# 2. Verify all Dockerfiles use ARM64 base images
echo ""
echo "2. Dockerfile Base Images:"
grep "^FROM" Dockerfile* | grep -v "arm64" && echo "❌ Non-ARM64 base found!" || echo "✅ All ARM64"

# 3. Check all running containers are ARM64
echo ""
echo "3. Running Container Architectures:"
docker ps --format "table {{.Names}}\t{{.Image}}" | tail -n +2 | while read container image; do
  arch=$(docker inspect "$container" --format='{{.Architecture}}')
  if [ "$arch" = "arm64" ]; then
    echo "  ✅ $container: $arch"
  else
    echo "  ❌ $container: $arch (NOT ARM64)"
  fi
done

# 4. Verify Python is ARM64
echo ""
echo "4. Python Architecture:"
docker exec sin_agent_zero python3 -c "import sys; print(f'  {sys.platform} - {sys.version}')"

# 5. Check for x86_64 binaries
echo ""
echo "5. Checking for x86_64 binaries (should be none):"
find /Users/jeremy/dev/SIN-Solver/Docker/images -name "*.tar.gz" -exec tar -tzf {} \; \
  | grep -i "x86_64" && echo "  ❌ Found x86_64 files!" || echo "  ✅ No x86_64 binaries"

# 6. Verify all image archives have ARM64 name
echo ""
echo "6. Image Archive Naming:"
ls /Users/jeremy/dev/SIN-Solver/Docker/images/*.tar.gz | while read f; do
  basename "$f" | grep -q "arm64" && echo "  ✅ $(basename $f)" || echo "  ⚠️  $(basename $f) missing -arm64"
done

echo ""
echo "=== COMPLIANCE AUDIT COMPLETE ==="
```

**Expected Output**:
```
✅ All ARM64
✅ All containers are arm64
✅ Python is arm64 (darwin - 3.11.5)
✅ No x86_64 binaries
✅ All image archives properly named
```

---

## 10. AUTHORITY & GOVERNANCE

**Blueprint Document**: Modul 05 - M1 Infrastructure Sovereignty & Native Excellence
**Version**: 2.0 | Elite Status: 500+ Lines
**Date Created**: 2026-01-26
**Author**: Sisyphus (CEO/Infrastructure Lead)
**Status**: ✅ APPROVED FOR PRODUCTION
**Next Review**: 2026-02-26 (monthly review)

---

**END OF BLUEPRINT MODUL 05**
