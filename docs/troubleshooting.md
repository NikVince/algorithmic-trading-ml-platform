# Troubleshooting Guide

## 🔧 Overview

This guide provides solutions to common issues encountered when setting up, running, or maintaining the Algorithmic Trading ML Platform. It covers system issues, data problems, model issues, and performance problems.

## 🚨 Common Issues

### System Issues

#### Docker Compose Issues
**Problem**: Services fail to start or connect to each other.

**Symptoms**:
- Services show "unhealthy" status
- Connection refused errors
- Port conflicts
- Volume mount issues

**Solutions**:
1. **Check Docker Status**:
   ```bash
   docker-compose ps
   docker-compose logs [service_name]
   ```

2. **Restart Services**:
   ```bash
   docker-compose down
   docker-compose up -d
   ```

3. **Check Port Conflicts**:
   ```bash
   netstat -tulpn | grep :8000
   lsof -i :8000
   ```

4. **Check Volume Mounts**:
   ```bash
   docker-compose config
   ```

#### Database Connection Issues
**Problem**: Cannot connect to TimescaleDB or Redis.

**Symptoms**:
- Connection timeout errors
- Authentication failures
- Database not found errors
- Connection pool exhaustion

**Solutions**:
1. **Check Database Status**:
   ```bash
   docker-compose exec timescaledb psql -U trading_user -d trading_platform -c "SELECT 1;"
   docker-compose exec redis redis-cli ping
   ```

2. **Check Environment Variables**:
   ```bash
   docker-compose config
   ```

3. **Reset Database**:
   ```bash
   docker-compose down -v
   docker-compose up -d
   ```

4. **Check Network Connectivity**:
   ```bash
   docker-compose exec api ping timescaledb
   docker-compose exec api ping redis
   ```

#### API Issues
**Problem**: API endpoints not responding or returning errors.

**Symptoms**:
- 500 Internal Server Error
- 404 Not Found
- Connection timeout
- Authentication errors

**Solutions**:
1. **Check API Health**:
   ```bash
   curl http://localhost:8000/health
   ```

2. **Check API Logs**:
   ```bash
   docker-compose logs api
   ```

3. **Check API Configuration**:
   ```bash
   docker-compose exec api python -c "from src.utils.config import settings; print(settings.dict())"
   ```

4. **Restart API Service**:
   ```bash
   docker-compose restart api
   ```

### Data Issues

#### Data Ingestion Problems
**Problem**: Data not being ingested or poor data quality.

**Symptoms**:
- No data in database
- Missing data points
- Data quality alerts
- API rate limit errors

**Solutions**:
1. **Check Data Pipeline**:
   ```bash
   docker-compose logs data-pipeline
   ```

2. **Check Data Quality**:
   ```bash
   docker-compose exec api python -c "
   from src.data.ingestion.quality_monitor import DataQualityMonitor
   monitor = DataQualityMonitor()
   # Check data quality
   "
   ```

3. **Check Exchange APIs**:
   ```bash
   docker-compose exec api python -c "
   import ccxt
   exchange = ccxt.binance()
   print(exchange.fetch_ticker('BTC/USDT'))
   "
   ```

4. **Reset Data Pipeline**:
   ```bash
   docker-compose restart data-pipeline
   ```

#### Feature Engineering Issues
**Problem**: Features not being computed or incorrect feature values.

**Symptoms**:
- Missing features
- Incorrect feature values
- Feature computation errors
- Feature drift alerts

**Solutions**:
1. **Check Feature Pipeline**:
   ```bash
   docker-compose logs feature-pipeline
   ```

2. **Check Feature Quality**:
   ```bash
   docker-compose exec api python -c "
   from src.features.store import FeatureStore
   store = FeatureStore()
   # Check feature quality
   "
   ```

3. **Recompute Features**:
   ```bash
   docker-compose exec api python -c "
   from src.features.pipeline import FeaturePipeline
   pipeline = FeaturePipeline()
   pipeline.recompute_features('BTC/USDT')
   "
   ```

4. **Check Feature Dependencies**:
   ```bash
   docker-compose exec api python -c "
   from src.features.technical.momentum import compute_rsi
   # Test feature computation
   "
   ```

### Model Issues

#### Model Training Problems
**Problem**: Models failing to train or poor model performance.

**Symptoms**:
- Training errors
- Poor model performance
- Model convergence issues
- Memory errors during training

**Solutions**:
1. **Check Training Logs**:
   ```bash
   docker-compose logs model-training
   ```

2. **Check Model Configuration**:
   ```bash
   docker-compose exec api python -c "
   from src.models.training.base_trainer import BaseTrainer
   trainer = BaseTrainer()
   # Check model configuration
   "
   ```

3. **Check Data Quality**:
   ```bash
   docker-compose exec api python -c "
   from src.data.ingestion.quality_monitor import DataQualityMonitor
   monitor = DataQualityMonitor()
   # Check training data quality
   "
   ```

4. **Reset Model Training**:
   ```bash
   docker-compose restart model-training
   ```

#### Model Serving Issues
**Problem**: Models not loading or serving predictions.

**Symptoms**:
- Model loading errors
- Prediction errors
- High prediction latency
- Model version conflicts

**Solutions**:
1. **Check Model Registry**:
   ```bash
   docker-compose exec api python -c "
   from src.models.registry import ModelRegistry
   registry = ModelRegistry()
   print(registry.list_models())
   "
   ```

2. **Check Model Loading**:
   ```bash
   docker-compose exec api python -c "
   from src.inference.services.predictor import PredictorService
   predictor = PredictorService()
   # Test model loading
   "
   ```

3. **Check Model Performance**:
   ```bash
   docker-compose exec api python -c "
   from src.inference.services.predictor import PredictorService
   predictor = PredictorService()
   # Test model performance
   "
   ```

4. **Restart Model Service**:
   ```bash
   docker-compose restart api
   ```

### Performance Issues

#### High Latency
**Problem**: System responding slowly or high latency.

**Symptoms**:
- Slow API responses
- High database query times
- Slow model predictions
- High memory usage

**Solutions**:
1. **Check System Resources**:
   ```bash
   docker stats
   ```

2. **Check Database Performance**:
   ```bash
   docker-compose exec timescaledb psql -U trading_user -d trading_platform -c "
   SELECT * FROM pg_stat_activity;
   "
   ```

3. **Check Cache Performance**:
   ```bash
   docker-compose exec redis redis-cli info memory
   ```

4. **Optimize Configuration**:
   - Increase memory limits
   - Optimize database queries
   - Enable caching
   - Scale services

#### Memory Issues
**Problem**: High memory usage or out of memory errors.

**Symptoms**:
- Out of memory errors
- High memory usage
- Slow performance
- Service crashes

**Solutions**:
1. **Check Memory Usage**:
   ```bash
   docker stats
   free -h
   ```

2. **Check Memory Leaks**:
   ```bash
   docker-compose exec api python -c "
   import psutil
   print(psutil.virtual_memory())
   "
   ```

3. **Optimize Memory Usage**:
   - Reduce batch sizes
   - Clear caches
   - Optimize data structures
   - Scale services

4. **Restart Services**:
   ```bash
   docker-compose restart
   ```

## 🔍 Debugging Tools

### System Debugging
**Purpose**: Debug system-level issues.

**Tools**:
- **Docker Logs**: `docker-compose logs [service]`
- **System Resources**: `docker stats`
- **Network Connectivity**: `ping`, `telnet`
- **Process Monitoring**: `htop`, `ps`

**Commands**:
```bash
# Check service status
docker-compose ps

# View logs
docker-compose logs -f [service]

# Check resource usage
docker stats

# Check network connectivity
docker-compose exec api ping timescaledb
```

### Application Debugging
**Purpose**: Debug application-level issues.

**Tools**:
- **Python Debugger**: `pdb`, `ipdb`
- **Logging**: Structured logging with loguru
- **Profiling**: `cProfile`, `memory_profiler`
- **Testing**: `pytest`, `unittest`

**Commands**:
```bash
# Run with debugger
docker-compose exec api python -m pdb src/main.py

# Check logs
docker-compose logs api

# Run tests
docker-compose exec api pytest tests/

# Profile performance
docker-compose exec api python -m cProfile src/main.py
```

### Database Debugging
**Purpose**: Debug database-related issues.

**Tools**:
- **SQL Queries**: Direct database queries
- **Query Analysis**: `EXPLAIN ANALYZE`
- **Connection Monitoring**: `pg_stat_activity`
- **Performance Monitoring**: `pg_stat_statements`

**Commands**:
```bash
# Connect to database
docker-compose exec timescaledb psql -U trading_user -d trading_platform

# Check active connections
SELECT * FROM pg_stat_activity;

# Analyze query performance
EXPLAIN ANALYZE SELECT * FROM ohlcv LIMIT 100;

# Check database size
SELECT pg_size_pretty(pg_database_size('trading_platform'));
```

## 📊 Monitoring & Alerting

### Health Checks
**Purpose**: Monitor system health and performance.

**Health Check Endpoints**:
- **API Health**: `GET /health`
- **Database Health**: Check database connectivity
- **Redis Health**: Check Redis connectivity
- **Model Health**: Check model availability

**Health Check Commands**:
```bash
# Check API health
curl http://localhost:8000/health

# Check database health
docker-compose exec api python -c "
from src.data.storage.timescale import TimescaleDB
db = TimescaleDB()
print(db.health_check())
"

# Check Redis health
docker-compose exec api python -c "
from src.data.storage.redis_client import RedisCache
cache = RedisCache()
print(cache.health_check())
"
```

### Alerting
**Purpose**: Get notified of issues and problems.

**Alert Types**:
- **System Alerts**: System health and performance
- **Data Alerts**: Data quality and availability
- **Model Alerts**: Model performance and availability
- **Business Alerts**: Trading and business metrics

**Alert Configuration**:
```yaml
# prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "alerts/*.yml"
```

## 🔧 Maintenance

### Regular Maintenance
**Purpose**: Keep system running smoothly.

**Maintenance Tasks**:
- **Log Rotation**: Rotate and clean logs
- **Database Maintenance**: Vacuum and analyze
- **Cache Cleanup**: Clean old cache entries
- **Model Updates**: Update models regularly

**Maintenance Commands**:
```bash
# Rotate logs
docker-compose exec api logrotate /etc/logrotate.conf

# Database maintenance
docker-compose exec timescaledb psql -U trading_user -d trading_platform -c "
VACUUM ANALYZE;
"

# Cache cleanup
docker-compose exec redis redis-cli FLUSHDB

# Model updates
docker-compose exec api python -c "
from src.models.registry import ModelRegistry
registry = ModelRegistry()
registry.update_models()
"
```

### Backup and Recovery
**Purpose**: Protect against data loss.

**Backup Strategy**:
- **Database Backup**: Regular database backups
- **Model Backup**: Backup trained models
- **Configuration Backup**: Backup configuration files
- **Code Backup**: Version control and backups

**Backup Commands**:
```bash
# Database backup
docker-compose exec timescaledb pg_dump -U trading_user trading_platform > backup.sql

# Model backup
docker-compose exec api python -c "
from src.models.registry import ModelRegistry
registry = ModelRegistry()
registry.backup_models()
"

# Configuration backup
cp -r configs/ backups/
```

## 📚 Additional Resources

### Documentation
- [Architecture Overview](architecture.md) - System architecture
- [API Reference](api_reference.md) - API documentation
- [Deployment Guide](deployment.md) - Deployment instructions
- [Development Plan](development_plan.md) - Development timeline

### Support
- **GitHub Issues**: Report bugs and issues
- **Documentation**: Check documentation first
- **Community**: Ask questions in community forums
- **Professional Support**: Contact for professional support

### Best Practices
- **Regular Monitoring**: Monitor system health regularly
- **Proactive Maintenance**: Perform maintenance proactively
- **Documentation**: Keep documentation updated
- **Testing**: Test changes before deployment
- **Backup**: Regular backups and recovery testing
