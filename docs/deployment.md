# Deployment Guide

## 🚀 Overview

This guide covers the complete deployment process for the Algorithmic Trading ML Platform, from local development to production environments.

## 📋 Prerequisites

### System Requirements
- **OS**: Linux (Ubuntu 22.04+), macOS (12.0+), or Windows 11+ with WSL2
- **RAM**: 8GB minimum, 16GB recommended
- **Storage**: 50GB free space
- **CPU**: 4 cores minimum, 8 cores recommended

### Software Dependencies
- **Docker**: 24.0+ with Docker Compose 2.20+
- **Python**: 3.11+ (for local development)
- **Git**: 2.30+
- **Make**: 4.0+ (optional, for convenience commands)

## 🏗️ Local Development Setup

### 1. Clone Repository
```bash
git clone https://github.com/your-username/algorithmic-trading-ml-platform.git
cd algorithmic-trading-ml-platform
```

### 2. Environment Configuration
```bash
# Copy environment template
cp .env.example .env

# Edit environment variables
nano .env
```

**Required Environment Variables:**
```bash
# Database Configuration
POSTGRES_DB=trading_platform
POSTGRES_USER=trading_user
POSTGRES_PASSWORD=secure_password
POSTGRES_HOST=timescaledb
POSTGRES_PORT=5432

# Redis Configuration
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=redis_password

# API Configuration
API_KEY=your_secure_api_key
SECRET_KEY=your_secret_key
ENVIRONMENT=development

# Exchange API Keys (for live trading)
BINANCE_API_KEY=your_binance_api_key
BINANCE_SECRET_KEY=your_binance_secret_key

# MLflow Configuration
MLFLOW_TRACKING_URI=http://mlflow:5000
MLFLOW_ARTIFACT_ROOT=/mlflow/artifacts
```

### 3. Start Services
```bash
# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f
```

### 4. Initialize Database
```bash
# Run database migrations
docker-compose exec api python scripts/setup_db.py

# Seed with sample data
docker-compose exec api python scripts/seed_data.py
```

### 5. Verify Installation
```bash
# Check API health
curl http://localhost:8000/health

# Check MLflow UI
open http://localhost:5000

# Check Grafana dashboard
open http://localhost:3000
```

## 🐳 Docker Configuration

### Docker Compose Services

```yaml
version: '3.8'

services:
  # TimescaleDB - Time-series database
  timescaledb:
    image: timescale/timescaledb:latest-pg14
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - timescale_data:/var/lib/postgresql/data
      - ./scripts/init_db.sql:/docker-entrypoint-initdb.d/init_db.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis - Caching and session storage
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # MLflow - Experiment tracking
  mlflow:
    image: python:3.11-slim
    command: mlflow server --backend-store-uri postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@timescaledb:5432/${POSTGRES_DB} --default-artifact-root /mlflow/artifacts --host 0.0.0.0
    ports:
      - "5000:5000"
    volumes:
      - mlflow_data:/mlflow/artifacts
    depends_on:
      - timescaledb
    environment:
      - MLFLOW_TRACKING_URI=http://mlflow:5000

  # FastAPI - Inference API
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - POSTGRES_HOST=timescaledb
      - REDIS_HOST=redis
      - MLFLOW_TRACKING_URI=http://mlflow:5000
    depends_on:
      - timescaledb
      - redis
      - mlflow
    volumes:
      - ./src:/app/src
      - ./configs:/app/configs

  # Streamlit - Dashboard
  dashboard:
    build:
      context: .
      dockerfile: docker/Dockerfile.dashboard
    ports:
      - "8501:8501"
    environment:
      - API_BASE_URL=http://api:8000
    depends_on:
      - api
    volumes:
      - ./src:/app/src

  # Prometheus - Metrics collection
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'

  # Grafana - Visualization
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources

volumes:
  timescale_data:
  redis_data:
  mlflow_data:
  prometheus_data:
  grafana_data:
```

### Dockerfile for API Service

```dockerfile
# docker/Dockerfile.api
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY configs/ ./configs/

# Set environment variables
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Start application
CMD ["uvicorn", "src.inference.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Dockerfile for Dashboard Service

```dockerfile
# docker/Dockerfile.dashboard
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/

# Set environment variables
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1

# Expose port
EXPOSE 8501

# Start application
CMD ["streamlit", "run", "src/dashboard/main.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

## 🌐 Production Deployment

### 1. Production Environment Setup

```bash
# Create production environment file
cp .env.example .env.production

# Edit production configuration
nano .env.production
```

**Production Environment Variables:**
```bash
# Database Configuration
POSTGRES_DB=trading_platform_prod
POSTGRES_USER=trading_user_prod
POSTGRES_PASSWORD=very_secure_password
POSTGRES_HOST=your-db-host.com
POSTGRES_PORT=5432

# Redis Configuration
REDIS_HOST=your-redis-host.com
REDIS_PORT=6379
REDIS_PASSWORD=very_secure_redis_password

# API Configuration
API_KEY=production_api_key
SECRET_KEY=production_secret_key
ENVIRONMENT=production
DEBUG=false

# Security
ALLOWED_HOSTS=your-domain.com,api.your-domain.com
CORS_ORIGINS=https://your-domain.com

# Monitoring
PROMETHEUS_ENDPOINT=http://prometheus:9090
GRAFANA_ENDPOINT=http://grafana:3000
```

### 2. Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  # Production API with load balancing
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    environment:
      - ENVIRONMENT=production
      - DEBUG=false
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Nginx load balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - api

  # Production database (external)
  # Configure connection to managed database service

  # Production Redis (external)
  # Configure connection to managed Redis service
```

### 3. Nginx Configuration

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream api_backend {
        server api:8000;
        server api:8000;
        server api:8000;
    }

    server {
        listen 80;
        server_name your-domain.com;

        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name your-domain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        location / {
            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            proxy_pass http://api_backend/health;
            access_log off;
        }
    }
}
```

## ☁️ Cloud Deployment

### AWS Deployment

#### 1. ECS Task Definition
```json
{
  "family": "trading-platform-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "your-account.dkr.ecr.region.amazonaws.com/trading-platform:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "API_KEY",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:api-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/trading-platform",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

#### 2. Application Load Balancer
```yaml
# terraform/alb.tf
resource "aws_lb" "trading_platform" {
  name               = "trading-platform-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = false
}

resource "aws_lb_target_group" "api" {
  name     = "trading-platform-api"
  port     = 8000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
    port                = "traffic-port"
    protocol            = "HTTP"
  }
}
```

### Google Cloud Platform Deployment

#### 1. Cloud Run Service
```yaml
# cloud-run.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: trading-platform-api
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "10"
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 100
      containers:
      - image: gcr.io/your-project/trading-platform:latest
        ports:
        - containerPort: 8000
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          limits:
            cpu: "2"
            memory: "4Gi"
```

#### 2. Cloud SQL Configuration
```yaml
# cloud-sql.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudsql-db-credentials
type: Opaque
data:
  username: <base64-encoded-username>
  password: <base64-encoded-password>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trading-platform-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: trading-platform-api
  template:
    metadata:
      labels:
        app: trading-platform-api
    spec:
      containers:
      - name: api
        image: gcr.io/your-project/trading-platform:latest
        env:
        - name: POSTGRES_HOST
          value: "127.0.0.1:5432"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: cloudsql-db-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cloudsql-db-credentials
              key: password
```

## 🔧 Configuration Management

### Environment-Specific Configs

```python
# src/utils/config.py
import os
from typing import Dict, Any
from pydantic import BaseSettings

class Settings(BaseSettings):
    # Database
    postgres_host: str = "localhost"
    postgres_port: int = 5432
    postgres_db: str = "trading_platform"
    postgres_user: str = "trading_user"
    postgres_password: str = ""
    
    # Redis
    redis_host: str = "localhost"
    redis_port: int = 6379
    redis_password: str = ""
    
    # API
    api_key: str = ""
    secret_key: str = ""
    environment: str = "development"
    debug: bool = True
    
    # Monitoring
    prometheus_endpoint: str = "http://localhost:9090"
    grafana_endpoint: str = "http://localhost:3000"
    
    class Config:
        env_file = ".env"
        case_sensitive = False

# Load environment-specific settings
def get_settings() -> Settings:
    env = os.getenv("ENVIRONMENT", "development")
    env_file = f".env.{env}" if env != "development" else ".env"
    
    return Settings(_env_file=env_file)

settings = get_settings()
```

### Feature Flags

```python
# src/utils/feature_flags.py
from typing import Dict, Any
import redis
import json

class FeatureFlags:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.cache_ttl = 300  # 5 minutes
    
    def is_enabled(self, flag_name: str, default: bool = False) -> bool:
        """Check if a feature flag is enabled"""
        try:
            cached = self.redis.get(f"feature_flag:{flag_name}")
            if cached:
                return json.loads(cached)
            
            # Default behavior
            return default
        except Exception:
            return default
    
    def set_flag(self, flag_name: str, enabled: bool, ttl: int = None):
        """Set a feature flag"""
        ttl = ttl or self.cache_ttl
        self.redis.setex(
            f"feature_flag:{flag_name}",
            ttl,
            json.dumps(enabled)
        )
```

## 📊 Monitoring Setup

### Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: 'trading-platform-api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: '/metrics'
    scrape_interval: 5s

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Trading Platform Overview",
    "panels": [
      {
        "title": "API Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Model Prediction Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(model_prediction_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      }
    ]
  }
}
```

## 🔒 Security Configuration

### SSL/TLS Setup

```bash
# Generate SSL certificates
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Configure certificate renewal
# Add to crontab
0 2 * * * /usr/bin/certbot renew --quiet
```

### Security Headers

```python
# src/inference/api/middleware/security.py
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.cors import CORSMiddleware

def setup_security_middleware(app: FastAPI):
    # CORS configuration
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["GET", "POST"],
        allow_headers=["*"],
    )
    
    # Trusted host middleware
    app.add_middleware(
        TrustedHostMiddleware,
        allowed_hosts=settings.allowed_hosts
    )
```

## 🚀 Deployment Scripts

### Automated Deployment

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=${1:-development}
VERSION=${2:-latest}

echo "Deploying to $ENVIRONMENT environment..."

# Build and push Docker images
docker build -t trading-platform:$VERSION .
docker tag trading-platform:$VERSION your-registry/trading-platform:$VERSION
docker push your-registry/trading-platform:$VERSION

# Deploy to environment
if [ "$ENVIRONMENT" = "production" ]; then
    # Production deployment
    kubectl apply -f k8s/production/
    kubectl rollout restart deployment/trading-platform-api
else
    # Development deployment
    docker-compose -f docker-compose.yml -f docker-compose.$ENVIRONMENT.yml up -d
fi

echo "Deployment completed successfully!"
```

### Health Check Script

```bash
#!/bin/bash
# scripts/health_check.sh

API_URL=${1:-http://localhost:8000}
TIMEOUT=${2:-30}

echo "Checking API health at $API_URL..."

# Wait for API to be ready
for i in $(seq 1 $TIMEOUT); do
    if curl -f -s "$API_URL/health" > /dev/null; then
        echo "✅ API is healthy!"
        exit 0
    fi
    echo "⏳ Waiting for API... ($i/$TIMEOUT)"
    sleep 1
done

echo "❌ API health check failed!"
exit 1
```

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Trading Platform

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      - name: Run tests
        run: pytest tests/ --cov=src/ --cov-report=xml
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: |
          docker build -t trading-platform:${{ github.sha }} .
          docker tag trading-platform:${{ github.sha }} trading-platform:latest
      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker push trading-platform:${{ github.sha }}
          docker push trading-platform:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # Add deployment commands here
```

## 📋 Troubleshooting

### Common Issues

#### 1. Database Connection Issues
```bash
# Check database connectivity
docker-compose exec api python -c "
import psycopg2
conn = psycopg2.connect(
    host='timescaledb',
    database='trading_platform',
    user='trading_user',
    password='secure_password'
)
print('Database connection successful!')
"
```

#### 2. Redis Connection Issues
```bash
# Check Redis connectivity
docker-compose exec api python -c "
import redis
r = redis.Redis(host='redis', port=6379, password='redis_password')
print('Redis connection successful!')
"
```

#### 3. Model Loading Issues
```bash
# Check model availability
curl -H "Authorization: Bearer $API_KEY" \
     http://localhost:8000/models
```

### Performance Optimization

#### 1. Database Optimization
```sql
-- Create indexes for better performance
CREATE INDEX CONCURRENTLY idx_ohlcv_symbol_time 
ON ohlcv (symbol, time DESC);

CREATE INDEX CONCURRENTLY idx_features_symbol_time 
ON features (symbol, timestamp DESC);
```

#### 2. Redis Optimization
```python
# Configure Redis for better performance
redis_config = {
    'maxmemory': '2gb',
    'maxmemory-policy': 'allkeys-lru',
    'save': '900 1 300 10 60 10000'
}
```

## 📚 Additional Resources

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [FastAPI Deployment](https://fastapi.tiangolo.com/deployment/)
- [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
