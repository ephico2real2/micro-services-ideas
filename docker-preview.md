# Dockerizing the Preview Generation Service

This guide walks through containerizing the Preview Generation Service using Docker with multi-stage builds and Docker Compose for orchestration.

## 1. Multi-Stage Dockerfile

First, create a `Dockerfile` in the root directory of your project:

```dockerfile
# ============ BUILD STAGE ============
FROM node:18-alpine AS builder

# Install build dependencies
RUN apk add --no-cache python3 make g++ git

# Set working directory
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# ============ RUNTIME STAGE ============
FROM node:18-alpine AS runtime

# Install runtime dependencies including requested tools
RUN apk add --no-cache ffmpeg nfs-utils ca-certificates curl \
    && apk add --no-cache mongodb-tools

# Create app directory
WORKDIR /app

# Create storage directories
RUN mkdir -p storage/local/previews storage/temp \
    && mkdir -p /mnt/nfs/previews \
    && chmod -R 755 storage /mnt/nfs

# Copy only production dependencies and app files from builder stage
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/config ./config
COPY --from=builder /app/src ./src
COPY --from=builder /app/server.js ./

# Non-root user for better security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs && \
    chown -R nodeuser:nodejs /app storage /mnt/nfs

# Switch to non-root user
USER nodeuser

# Expose API port (will be mapped to localhost only in docker-compose)
EXPOSE 3001

# Set environmental variables
ENV NODE_ENV=production

# Start the service
CMD ["node", "server.js"]
```

## 2. Docker Compose Configuration

Create a `docker-compose.yml` file in the root directory:

```yaml
version: '3.8'

services:
  # MongoDB database
  mongodb:
    image: mongo:6.0
    container_name: preview-mongodb
    restart: unless-stopped
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_USERNAME:-admin}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD:-password}
    volumes:
      - mongodb_data:/data/db
    ports:
      - "127.0.0.1:27017:27017"
    networks:
      - preview-network
    healthcheck:
      test: mongosh --eval 'db.runCommand("ping").ok' --quiet
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

  # Preview generation service
  preview-service:
    container_name: preview-service
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    restart: unless-stopped
    depends_on:
      mongodb:
        condition: service_healthy
    environment:
      - NODE_ENV=production
      - PORT=3001
      - MONGODB_URI=mongodb://${MONGO_USERNAME:-admin}:${MONGO_PASSWORD:-password}@mongodb:27017/preview-service?authSource=admin
      - STORAGE_TYPE=${STORAGE_TYPE:-local}
      # S3 configuration (if using S3 storage)
      - S3_ACCESS_KEY=${S3_ACCESS_KEY:-}
      - S3_SECRET_KEY=${S3_SECRET_KEY:-}
      - S3_BUCKET=${S3_BUCKET:-}
      - S3_REGION=${S3_REGION:-}
      - S3_ENDPOINT=${S3_ENDPOINT:-}
      # NFS configuration (if using NFS storage)
      - NFS_SERVER=${NFS_SERVER:-}
      - NFS_EXPORT_PATH=${NFS_EXPORT_PATH:-}
      - NFS_MOUNT_PATH=/mnt/nfs/previews
    ports:
      - "127.0.0.1:3001:3001"
    volumes:
      - ./storage:/app/storage
      # For NFS, uncomment the next line and configure docker-compose.override.yml
      # - nfs-mount:/mnt/nfs/previews
    networks:
      - preview-network

  # MinIO service (S3-compatible storage for testing)
  minio:
    image: minio/minio
    container_name: preview-minio
    restart: unless-stopped
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER:-minioadmin}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-minioadmin}
    command: server /data --console-address ":9001"
    ports:
      - "127.0.0.1:9000:9000"
      - "127.0.0.1:9001:9001"
    volumes:
      - minio_data:/data
    networks:
      - preview-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  mongodb_data:
  minio_data:
  # Uncomment for NFS volume if needed
  # nfs-mount:
  #   driver: local
  #   driver_opts:
  #     type: "nfs"
  #     o: "addr=${NFS_SERVER},nolock,soft,rw"
  #     device: ":${NFS_EXPORT_PATH}"

networks:
  preview-network:
    driver: bridge
```

## 3. Environment Configuration

Create a `.env` file for configuring your services:

```env
# MongoDB Configuration
MONGO_USERNAME=admin
MONGO_PASSWORD=password

# Storage Configuration (local, s3, nfs)
STORAGE_TYPE=local

# S3 Configuration (if using S3 storage)
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=previews
S3_ENDPOINT=http://minio:9000
S3_REGION=us-east-1

# NFS Configuration (if using NFS storage)
NFS_SERVER=192.168.1.100
NFS_EXPORT_PATH=/exports/media
```

## 4. Create .dockerignore

Create a `.dockerignore` file to exclude unnecessary files:

```
node_modules
npm-debug.log
.git
.github
.vscode
.env
storage
logs
*.md
```

## 5. Running the Application

### Build and Start Services

```bash
# Build and start all services
docker-compose up -d --build

# View logs
docker-compose logs -f preview-service
```

### Initialize MinIO (if using S3 storage)

If you're using S3 storage with MinIO, you need to create a bucket:

```bash
# Install MinIO client
docker run --rm -it --entrypoint=/bin/sh minio/mc

# Configure MinIO client
mc alias set local http://localhost:9000 minioadmin minioadmin

# Create bucket
mc mb local/previews
```

Alternatively, you can use the MinIO web console at http://localhost:9001 (login with minioadmin/minioadmin).

### Test the Service

```bash
# Upload a test video and generate previews
curl -X POST \
  http://localhost:3001/api/preview/generate \
  -H 'Content-Type: multipart/form-data' \
  -F 'source=@/path/to/test-video.mp4' \
  -F 'sourceId=test-123' \
  -F 'userId=test-user' \
  -F 'previewCount=3' \
  -F 'duration=5' \
  -F 'quality=medium'

# Check service health
curl http://localhost:3001/api/health
```

## 6. Managing the Application

### Using Different Storage Backends

To switch between storage backends, update the `STORAGE_TYPE` in your `.env` file:

#### Local Storage (Default)
```env
STORAGE_TYPE=local
```

#### S3 Storage with MinIO
```env
STORAGE_TYPE=s3
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=previews
S3_ENDPOINT=http://minio:9000
```

#### NFS Storage
For NFS storage:
1. Uncomment the `nfs-mount` volume in `docker-compose.yml`
2. Configure NFS settings in `.env`:
   ```env
   STORAGE_TYPE=nfs
   NFS_SERVER=192.168.1.100
   NFS_EXPORT_PATH=/exports/media
   ```

### Useful Docker Commands

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (WARNING: deletes all data)
docker-compose down -v

# View container logs
docker-compose logs -f

# Connect to MongoDB using mongosh
docker exec -it preview-mongodb mongosh -u admin -p password

# Check container status
docker-compose ps

# View resource usage
docker stats
```

## 7. Accessing Container Tools

The runtime container includes several tools that can be accessed via `docker exec`:

```bash
# Access bash in the container
docker exec -it preview-service /bin/sh

# Use curl from the container
docker exec -it preview-service curl -s http://localhost:3001/api/health

# Use mongosh to connect to MongoDB
docker exec -it preview-service mongosh mongodb://admin:password@mongodb:27017/preview-service?authSource=admin
```

## 8. Troubleshooting

### Container Won't Start

Check logs for errors:
```bash
docker-compose logs preview-service
```

### Storage Connection Issues

For S3 issues:
```bash
# Test S3 connection from container
docker exec -it preview-service curl -I http://minio:9000
```

For NFS issues:
```bash
# Check NFS mount
docker exec -it preview-service df -h
```

### MongoDB Connection Issues

```bash
# Test MongoDB connection
docker exec -it preview-service mongosh --eval 'db.runCommand("ping").ok' mongodb://admin:password@mongodb:27017/preview-service?authSource=admin
```

## 9. Production Considerations

For production deployment:

1. Use secrets management instead of environment variables
2. Configure proper resource limits
3. Set up a reverse proxy with SSL/TLS
4. Implement monitoring and alerting
5. Use Docker Swarm or Kubernetes for orchestration
6. Set up backup and restore procedures
7. Implement log aggregation

This concludes the guide for dockerizing the Preview Generation Service.
