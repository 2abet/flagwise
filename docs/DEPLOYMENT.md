# Deployment Guide

This guide covers production deployment strategies for FlagWise, including security hardening, scaling considerations, and operational best practices.

## Production Architecture

### Recommended Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Load Balancer │    │   Web Frontend  │    │   API Backend   │
│   (nginx/ALB)   │────│   (React App)   │────│   (FastAPI)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Kafka Cluster │    │   PostgreSQL    │    │   Redis Cache   │
│   (Data Source) │    │   (Primary DB)  │    │   (Sessions)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Component Scaling
- **Web Frontend**: Stateless, can run multiple instances
- **API Backend**: Horizontally scalable with load balancing
- **Database**: Single primary with read replicas
- **Kafka Consumer**: Can run multiple instances for throughput

## Docker Production Setup

### Production Docker Compose
Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  web:
    build:
      context: ./services/web
      dockerfile: Dockerfile.prod
    ports:
      - "80:80"
      - "443:443"
    environment:
      - NODE_ENV=production
      - REACT_APP_API_URL=https://api.yourdomain.com
    volumes:
      - ./ssl:/etc/ssl/certs
    depends_on:
      - api

  api:
    build:
      context: ./services/api
      dockerfile: Dockerfile.prod
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgresql://user:pass@db:5432/flagwise
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
    depends_on:
      - db
      - redis
    volumes:
      - ./logs:/app/logs

  consumer:
    build:
      context: ./services/consumer
      dockerfile: Dockerfile.prod
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgresql://user:pass@db:5432/flagwise
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      - db
      - kafka

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=flagwise
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

volumes:
  postgres_data:
  redis_data:
```

### Environment Configuration
⚠️ **Security Warning**: Do NOT commit `.env.prod` to version control. For production, use Kubernetes Secrets, environment variable injection from secure vaults (AWS Secrets Manager, HashiCorp Vault), or container orchestration platform secrets management.

Create `.env.prod`:

```bash
# Database
DB_USER=flagwise_prod
DB_PASSWORD=secure_random_password_here
DATABASE_URL=postgresql://flagwise_prod:secure_password@db:5432/flagwise

# Security
JWT_SECRET_KEY=your_jwt_secret_key_here
ENCRYPTION_KEY=your_32_byte_encryption_key_here
REDIS_PASSWORD=secure_redis_password

# Application
ENVIRONMENT=production
LOG_LEVEL=INFO
CORS_ORIGINS=https://yourdomain.com

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
KAFKA_TOPIC=llm_requests

# Monitoring
SENTRY_DSN=your_sentry_dsn_here
```

## Kubernetes Deployment

### Namespace and ConfigMap
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flagwise

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: flagwise-config
  namespace: flagwise
data:
  DATABASE_URL: "postgresql://user:pass@postgres:5432/flagwise"
  KAFKA_BOOTSTRAP_SERVERS: "kafka:9092"
  ENVIRONMENT: "production"
```

### API Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flagwise-api
  namespace: flagwise
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flagwise-api
  template:
    metadata:
      labels:
        app: flagwise-api
    spec:
      containers:
      - name: api
        image: flagwise/api:latest
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: flagwise-config
        - secretRef:
            name: flagwise-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: flagwise-api-service
  namespace: flagwise
spec:
  selector:
    app: flagwise-api
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

### Web Frontend Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flagwise-web
  namespace: flagwise
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flagwise-web
  template:
    metadata:
      labels:
        app: flagwise-web
    spec:
      containers:
      - name: web
        image: flagwise/web:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"

---
apiVersion: v1
kind: Service
metadata:
  name: flagwise-web-service
  namespace: flagwise
spec:
  selector:
    app: flagwise-web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

## Security Hardening

### SSL/TLS Configuration
```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/ssl/certs/yourdomain.crt;
    ssl_certificate_key /etc/ssl/private/yourdomain.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://flagwise-web:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://flagwise-api:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Database Security
```sql
-- Create dedicated database user
CREATE USER flagwise_prod WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE flagwise TO flagwise_prod;
GRANT USAGE ON SCHEMA public TO flagwise_prod;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO flagwise_prod;

-- Enable SSL connections only
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = 'server.crt';
ALTER SYSTEM SET ssl_key_file = 'server.key';
```

### Network Security
```yaml
# Network policies for Kubernetes
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: flagwise-network-policy
  namespace: flagwise
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

## Monitoring and Observability

### Health Checks
```python
# API health endpoint
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow(),
        "version": "1.0.0",
        "database_connected": await check_database_connection(),
        "kafka_connected": await check_kafka_connection()
    }
```

### Logging Configuration
```yaml
# Fluentd configuration for log aggregation
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/flagwise-*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      format json
    </source>
    
    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch.logging.svc.cluster.local
      port 9200
      index_name flagwise-logs
    </match>
```

### Metrics Collection
```python
# Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge

request_count = Counter('flagwise_requests_total', 'Total requests processed')
request_duration = Histogram('flagwise_request_duration_seconds', 'Request duration')
active_sessions = Gauge('flagwise_active_sessions', 'Number of active sessions')
```

## Backup and Disaster Recovery

### Database Backup Strategy
```bash
#!/bin/bash
# Automated backup script

BACKUP_DIR="/backups/flagwise"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="flagwise"

# Create backup directory
mkdir -p $BACKUP_DIR

# Full database backup
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME > $BACKUP_DIR/flagwise_$DATE.sql

# Compress backup
gzip $BACKUP_DIR/flagwise_$DATE.sql

# Upload to S3
aws s3 cp $BACKUP_DIR/flagwise_$DATE.sql.gz s3://flagwise-backups/

# Cleanup old backups (keep 30 days)
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
```

### Disaster Recovery Plan
1. **Database Recovery**: Restore from latest backup
2. **Application Recovery**: Redeploy from container registry
3. **Data Validation**: Verify data integrity post-recovery
4. **Service Verification**: Run health checks and smoke tests

## Performance Optimization

### Database Tuning
```sql
-- PostgreSQL configuration for production
shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
```

### Application Scaling
```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flagwise-api-hpa
  namespace: flagwise
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flagwise-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Maintenance Procedures

### Rolling Updates
```bash
# Update API deployment
kubectl set image deployment/flagwise-api api=flagwise/api:v1.1.0 -n flagwise

# Monitor rollout
kubectl rollout status deployment/flagwise-api -n flagwise

# Rollback if needed
kubectl rollout undo deployment/flagwise-api -n flagwise
```

### Database Maintenance
```sql
-- Weekly maintenance tasks
VACUUM ANALYZE;
REINDEX DATABASE flagwise;
UPDATE pg_stat_statements_reset();
```

### Log Rotation
```bash
# Logrotate configuration
/var/log/flagwise/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 644 flagwise flagwise
    postrotate
        systemctl reload flagwise-api
    endscript
}
```

## Troubleshooting

### Common Issues
- **High CPU usage**: Check for inefficient queries, scale horizontally
- **Memory leaks**: Monitor application metrics, restart if needed
- **Database locks**: Identify and terminate long-running queries
- **Network connectivity**: Verify service discovery and DNS resolution

### Diagnostic Commands
```bash
# Check pod status
kubectl get pods -n flagwise

# View logs
kubectl logs -f deployment/flagwise-api -n flagwise

# Check resource usage
kubectl top pods -n flagwise

# Database connections
kubectl exec -it postgres-pod -- psql -c "SELECT * FROM pg_stat_activity;"
```

## Security Checklist

- [ ] SSL/TLS certificates configured and valid
- [ ] Database connections encrypted
- [ ] Secrets stored in secure secret management
- [ ] Network policies implemented
- [ ] Regular security updates applied
- [ ] Access logs monitored
- [ ] Backup encryption enabled
- [ ] Vulnerability scanning automated
- [ ] Rate limiting configured
- [ ] CORS properly restricted
- [ ] Database connection pooling configured
- [ ] SQL injection prevention verified
- [ ] Security headers configured (CSP, X-Frame-Options, etc.)
- [ ] Regular security scanning/vulnerability assessments scheduled
- [ ] Incident response plan documented
- [ ] Access control audits scheduled