# Deployment & Operations

## Overview

This document outlines deployment procedures, operational processes, monitoring, and maintenance for the Virtual Receptionist system.

---

## Infrastructure Setup

### DigitalOcean Environment Configuration

#### 1. Initial Setup

```bash
# Install doctl (DigitalOcean CLI)
brew install doctl  # macOS
# or
snap install doctl  # Linux

# Authenticate
doctl auth init

# Create project
doctl projects create \
  --name "Virtual Receptionist" \
  --description "AI-powered hotel receptionist" \
  --purpose "Web Application"
```

#### 2. Create Kubernetes Cluster

```bash
# Create cluster (production)
doctl kubernetes cluster create virtual-receptionist-prod \
  --region fra1 \
  --version 1.28.2-do.0 \
  --node-pool "name=workers;size=s-2vcpu-4gb;count=3;auto-scale=true;min-nodes=3;max-nodes=10" \
  --ha

# Get kubeconfig
doctl kubernetes cluster kubeconfig save virtual-receptionist-prod

# Verify connection
kubectl cluster-info
```

#### 3. Provision Managed Databases

```bash
# PostgreSQL
doctl databases create virtual-receptionist-db \
  --engine pg \
  --version 15 \
  --region fra1 \
  --size db-s-2vcpu-4gb \
  --num-nodes 1

# Redis
doctl databases create virtual-receptionist-cache \
  --engine redis \
  --version 7 \
  --region fra1 \
  --size db-s-1vcpu-1gb
```

#### 4. Create Object Storage

```bash
# Create Spaces bucket
doctl compute cdn create \
  --origin "virtual-receptionist.fra1.digitaloceanspaces.com"

# Configure CORS
cat > cors.json <<EOF
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://app.virtualreceptionist.bg"],
      "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000
    }
  ]
}
EOF

s3cmd setcors cors.json s3://virtual-receptionist
```

---

## Application Deployment

### Kubernetes Configuration

#### Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: virtual-receptionist
```

#### ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: virtual-receptionist
data:
  NODE_ENV: "production"
  DATABASE_HOST: "db-postgresql-fra1-12345.b.db.ondigitalocean.com"
  REDIS_HOST: "db-redis-fra1-67890.b.db.ondigitalocean.com"
  TWILIO_WEBHOOK_URL: "https://api.virtualreceptionist.bg/webhooks/twilio"
```

#### Secrets

```yaml
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: virtual-receptionist
type: Opaque
stringData:
  DATABASE_PASSWORD: "${DATABASE_PASSWORD}"
  JWT_SECRET: "${JWT_SECRET}"
  TWILIO_AUTH_TOKEN: "${TWILIO_AUTH_TOKEN}"
  OPENAI_API_KEY: "${OPENAI_API_KEY}"
  GOOGLE_CLOUD_API_KEY: "${GOOGLE_CLOUD_API_KEY}"
```

#### API Deployment

```yaml
# k8s/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: virtual-receptionist
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: registry.digitalocean.com/virtual-receptionist/api:latest
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: virtual-receptionist
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
```

#### Frontend Deployment

```yaml
# k8s/frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: virtual-receptionist
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: registry.digitalocean.com/virtual-receptionist/frontend:latest
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: virtual-receptionist
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
```

#### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: virtual-receptionist
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.virtualreceptionist.bg
    - api.virtualreceptionist.bg
    secretName: tls-certificate
  rules:
  - host: app.virtualreceptionist.bg
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: api.virtualreceptionist.bg
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
```

#### HorizontalPodAutoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: virtual-receptionist
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
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

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

env:
  REGISTRY: registry.digitalocean.com
  IMAGE_NAME: virtual-receptionist

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Run integration tests
        run: npm run test:integration

      - name: Check coverage
        run: npm run test:coverage

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker image
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }} ./backend
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }} ./frontend

      - name: Login to DigitalOcean Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
          password: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Push images
        run: |
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }}

          docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }} ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:latest
          docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }} ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:latest

          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:latest
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Save kubeconfig
        run: doctl kubernetes cluster kubeconfig save virtual-receptionist-prod

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/api api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }} -n virtual-receptionist
          kubectl set image deployment/frontend frontend=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }} -n virtual-receptionist

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/api -n virtual-receptionist
          kubectl rollout status deployment/frontend -n virtual-receptionist

      - name: Run smoke tests
        run: |
          curl -f https://api.virtualreceptionist.bg/health || exit 1
          curl -f https://app.virtualreceptionist.bg || exit 1

      - name: Notify on failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployment failed!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Database Migrations

### Migration Strategy

**Tool**: TypeORM Migrations

```typescript
// src/database/migrations/1700000000000-InitialSchema.ts
import { MigrationInterface, QueryRunner } from 'typeorm'

export class InitialSchema1700000000000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE properties (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
      )
    `)

    await queryRunner.query(`
      CREATE TABLE rooms (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        property_id UUID NOT NULL REFERENCES properties(id),
        room_number VARCHAR(20) NOT NULL,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
      )
    `)

    // ... more tables
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE rooms`)
    await queryRunner.query(`DROP TABLE properties`)
  }
}
```

### Running Migrations

```bash
# Generate migration
npm run migration:generate -- -n CreateBookingsTable

# Run migrations
npm run migration:run

# Revert last migration
npm run migration:revert

# Production migration (with backup)
# 1. Backup database
doctl databases backups create virtual-receptionist-db

# 2. Run migration in transaction
npm run migration:run

# 3. Verify data integrity
npm run db:verify

# 4. If failed, restore from backup
doctl databases backups restore virtual-receptionist-db backup-id
```

---

## Monitoring

### Prometheus + Grafana

#### Prometheus Configuration

```yaml
# k8s/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
```

#### Custom Metrics

```typescript
// src/monitoring/metrics.ts
import { Registry, Counter, Histogram, Gauge } from 'prom-client'

const register = new Registry()

// Counters
export const callsTotal = new Counter({
  name: 'calls_total',
  help: 'Total number of calls received',
  labelNames: ['property_id', 'outcome'],
  registers: [register]
})

export const bookingsCreated = new Counter({
  name: 'bookings_created_total',
  help: 'Total bookings created',
  labelNames: ['property_id', 'source'],
  registers: [register]
})

// Histograms
export const callDuration = new Histogram({
  name: 'call_duration_seconds',
  help: 'Call duration in seconds',
  labelNames: ['property_id'],
  buckets: [30, 60, 120, 180, 300, 600],
  registers: [register]
})

export const apiResponseTime = new Histogram({
  name: 'api_response_time_seconds',
  help: 'API response time',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5],
  registers: [register]
})

// Gauges
export const activeCalls = new Gauge({
  name: 'active_calls',
  help: 'Number of currently active calls',
  labelNames: ['property_id'],
  registers: [register]
})

export const aiServiceLatency = new Gauge({
  name: 'ai_service_latency_ms',
  help: 'Latency to AI services',
  labelNames: ['service'], // 'stt', 'tts', 'llm'
  registers: [register]
})

// Middleware
export function metricsMiddleware(req, res, next) {
  const start = Date.now()

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000

    apiResponseTime.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
        status: res.statusCode
      },
      duration
    )
  })

  next()
}

// Expose metrics endpoint
export function metricsHandler(req, res) {
  res.set('Content-Type', register.contentType)
  res.end(register.metrics())
}
```

### Grafana Dashboards

**Dashboard: Call Metrics**

```json
{
  "dashboard": {
    "title": "Virtual Receptionist - Calls",
    "panels": [
      {
        "title": "Calls per Hour",
        "targets": [
          {
            "expr": "rate(calls_total[1h])"
          }
        ]
      },
      {
        "title": "Call Outcomes",
        "targets": [
          {
            "expr": "sum by (outcome) (calls_total)"
          }
        ]
      },
      {
        "title": "Average Call Duration",
        "targets": [
          {
            "expr": "avg(call_duration_seconds)"
          }
        ]
      },
      {
        "title": "Active Calls",
        "targets": [
          {
            "expr": "active_calls"
          }
        ]
      }
    ]
  }
}
```

### Alerting Rules

```yaml
# prometheus-rules.yaml
groups:
  - name: virtual_receptionist
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(calls_total{outcome="error"}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }}% over the last 5 minutes"

      - alert: SlowAPIResponse
        expr: histogram_quantile(0.95, api_response_time_seconds) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API response time degraded"
          description: "95th percentile response time is {{ $value }}s"

      - alert: AIServiceDown
        expr: up{job="ai-service"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "AI service is down"
          description: "AI service has been down for 2 minutes"

      - alert: DatabaseConnectionPoolExhausted
        expr: database_connections_active / database_connections_max > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool near capacity"
```

---

## Logging

### Log Aggregation

**Stack**: Loki + Promtail

```yaml
# k8s/promtail-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  promtail.yml: |
    server:
      http_listen_port: 9080

    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
```

### Structured Logging

```typescript
// src/utils/logger.ts
import winston from 'winston'

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'virtual-receptionist',
    environment: process.env.NODE_ENV
  },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    new winston.transports.File({
      filename: 'logs/combined.log'
    })
  ]
})

// Usage
logger.info('Call started', {
  callId: 'call-123',
  propertyId: 'property-456',
  caller: '+359888123456'
})

logger.error('Booking creation failed', {
  error: error.message,
  stack: error.stack,
  bookingData: { /* ... */ }
})
```

---

## Backup & Disaster Recovery

### Database Backups

**Automated Backups** (DigitalOcean Managed):
- Daily automatic backups
- 7-day retention
- Point-in-time recovery (up to 7 days)

**Manual Backups**:
```bash
# Create backup
doctl databases backups create virtual-receptionist-db

# List backups
doctl databases backups list virtual-receptionist-db

# Restore from backup
doctl databases backups restore virtual-receptionist-db backup-id
```

### Application State Backup

```bash
# Backup Kubernetes resources
kubectl get all -n virtual-receptionist -o yaml > backup-k8s-resources.yaml

# Backup secrets (encrypted)
kubectl get secrets -n virtual-receptionist -o yaml | \
  gpg --encrypt > backup-secrets.yaml.gpg
```

### Disaster Recovery Plan

**RTO (Recovery Time Objective)**: 4 hours
**RPO (Recovery Point Objective)**: 1 hour

**Recovery Steps**:

1. **Detect Failure** (Target: 5 minutes)
   - Automated monitoring alerts
   - Verify scope of failure

2. **Activate DR Plan** (Target: 15 minutes)
   - Notify team
   - Assess situation
   - Decide on recovery strategy

3. **Restore Services** (Target: 2 hours)
   ```bash
   # Restore database from latest backup
   doctl databases backups restore virtual-receptionist-db latest

   # Redeploy applications
   kubectl apply -f k8s/

   # Verify health
   kubectl get pods -n virtual-receptionist
   curl https://api.virtualreceptionist.bg/health
   ```

4. **Verify Functionality** (Target: 1 hour)
   - Run smoke tests
   - Check recent bookings
   - Test call flows
   - Verify integrations

5. **Post-Incident Review** (Within 24 hours)
   - Document incident
   - Identify root cause
   - Implement preventive measures

---

## Maintenance

### Routine Maintenance Tasks

**Daily**:
- Check error logs
- Review active alerts
- Monitor resource usage
- Verify backup completion

**Weekly**:
- Review performance metrics
- Analyze cost trends
- Update dependencies (security patches)
- Review and address low-priority bugs

**Monthly**:
- Security audit
- Cost optimization review
- Capacity planning
- Update documentation

**Quarterly**:
- Major dependency updates
- Performance testing
- Disaster recovery drill
- Review and update runbooks

### Scaling Operations

**Scale Up** (anticipated high load):
```bash
# Increase replicas
kubectl scale deployment api --replicas=10 -n virtual-receptionist

# Upgrade database
doctl databases resize virtual-receptionist-db \
  --size db-s-4vcpu-8gb \
  --num-nodes 2
```

**Scale Down** (reduce costs):
```bash
# Decrease replicas
kubectl scale deployment api --replicas=3 -n virtual-receptionist
```

---

## Runbooks

### Runbook: High Error Rate

**Symptoms**:
- Error rate >5%
- Alert: HighErrorRate

**Investigation**:
1. Check error logs:
   ```bash
   kubectl logs -l app=api -n virtual-receptionist --tail=100 | grep ERROR
   ```

2. Check external service status:
   - Twilio Status: https://status.twilio.com
   - OpenAI Status: https://status.openai.com
   - Google Cloud Status: https://status.cloud.google.com

3. Check database connections:
   ```bash
   kubectl exec -it deployment/api -n virtual-receptionist -- \
     psql -h $DB_HOST -U $DB_USER -c "SELECT count(*) FROM pg_stat_activity"
   ```

**Resolution**:
- If external service down: Wait for recovery, notify customers
- If database issue: Restart pods, check connection pool settings
- If application bug: Rollback to previous version

### Runbook: Database Connection Pool Exhausted

**Symptoms**:
- Slow response times
- Connection timeout errors

**Investigation**:
```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';

-- Check long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

**Resolution**:
```bash
# Increase connection pool size
kubectl set env deployment/api \
  DATABASE_POOL_SIZE=50 \
  -n virtual-receptionist

# Kill long-running queries (if necessary)
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 minutes';
```

---

## Security Operations

### Security Monitoring

**SIEM Integration**: Forward logs to security monitoring

**Automated Scans**:
```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
```

### Incident Response

**Security Incident Checklist**:
1. ✅ Isolate affected systems
2. ✅ Preserve evidence (logs, snapshots)
3. ✅ Assess scope and impact
4. ✅ Notify stakeholders
5. ✅ Remediate vulnerability
6. ✅ Restore services
7. ✅ Post-incident review
8. ✅ Update security measures

---

## Cost Monitoring

### DigitalOcean Cost Tracking

```bash
# Get current month usage
doctl invoice list

# Get resource costs
doctl balance get

# Export cost report
doctl invoice get INVOICE_ID --format json > cost-report.json
```

### Cost Optimization

**Review Monthly**:
- Unused resources (test databases, old droplets)
- Over-provisioned resources
- Storage cleanup (old backups, logs, recordings)
- AI service usage optimization

**Automated Cleanup**:
```bash
# Delete old call recordings (>90 days)
s3cmd ls s3://virtual-receptionist/recordings/ | \
  awk '{if ($1 < (systime() - 7776000)) print $4}' | \
  xargs -I {} s3cmd del {}

# Delete old log files
find /var/log -name "*.log" -mtime +30 -delete
```

---

## Next Steps

1. Review [Implementation Roadmap](./10-implementation-roadmap.md) for deployment timeline
2. See [Testing Strategy](./12-testing-strategy.md) for testing requirements before deployment
3. Check [Security & Compliance](./09-security-compliance.md) for security operations
4. Review [Cost Analysis](./11-cost-analysis.md) for budget planning

---

**End of Documentation Suite**
