# Apache Superset - Deployment & Dashboard Guide

## 📋 Daftar Isi
- [Overview Apache Superset](#overview-apache-superset)
- [Deployment Methods](#deployment-methods)
- [Database Configuration](#database-configuration)
- [Dashboard Creation](#dashboard-creation)
- [Advanced Features](#advanced-features)
- [Best Practices](#best-practices)
- [Maintenance & Monitoring](#maintenance--monitoring)

## 🚀 Overview Apache Superset

### Apa itu Apache Superset?
Apache Superset adalah open-source business intelligence web application yang memungkinkan:
- **Visualisasi Data** yang powerful dan customizable
- **SQL Lab** untuk query writing dan exploration
- **Dashboard** yang interactive dan responsive
- **Enterprise-grade** security dan authentication
- **Scalable** architecture untuk large datasets

### Keunggulan vs Metabase
- ✅ Lebih powerful untuk large datasets
- ✅ Custom visualization capabilities
- ✅ Better enterprise features
- ✅ More flexible deployment options
- ✅ Advanced caching dan performance

## 🛠 Deployment Methods

### 1. Deployment dengan Docker Compose (Recommended)

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgresql:
    image: postgres:13
    container_name: superset_postgres
    environment:
      - POSTGRES_DB=superset
      - POSTGRES_USER=superset
      - POSTGRES_PASSWORD=superset_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - superset-network

  redis:
    image: redis:6-alpine
    container_name: superset_redis
    networks:
      - superset-network

  superset:
    image: apache/superset:latest
    container_name: superset
    hostname: superset
    ports:
      - "8088:8088"
    environment:
      - SUPERSET_SECRET_KEY=your-secret-key-here
      - DATABASE_DB=superset
      - DATABASE_HOST=postgresql
      - DATABASE_PASSWORD=superset_pass
      - DATABASE_USER=superset
      - REDIS_HOST=redis
    volumes:
      - superset_data:/app/superset_home
    depends_on:
      - postgresql
      - redis
    networks:
      - superset-network
    restart: unless-stopped

  superset-init:
    image: apache/superset:latest
    container_name: superset_init
    depends_on:
      - superset
    command: >
      bash -c "
        superset fab create-admin &&
        superset db upgrade &&
        superset init &&
        superset load_examples"
    environment:
      - SUPERSET_SECRET_KEY=your-secret-key-here
      - DATABASE_DB=superset
      - DATABASE_HOST=postgresql
      - DATABASE_PASSWORD=superset_pass
      - DATABASE_USER=superset
    volumes:
      - superset_data:/app/superset_home
    networks:
      - superset-network

volumes:
  postgres_data:
  superset_data:

networks:
  superset-network:
    driver: bridge
```

### 2. Deployment dengan Kubernetes

```yaml
# superset-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: superset
  namespace: analytics
spec:
  replicas: 2
  selector:
    matchLabels:
      app: superset
  template:
    metadata:
      labels:
        app: superset
    spec:
      containers:
      - name: superset
        image: apache/superset:latest
        ports:
        - containerPort: 8088
        env:
        - name: SUPERSET_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: superset-secrets
              key: secret-key
        - name: DATABASE_HOST
          value: "postgresql-service"
        - name: DATABASE_USER
          value: "superset"
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: superset-secrets
              key: database-password
        - name: REDIS_HOST
          value: "redis-service"
        volumeMounts:
        - name: superset-config
          mountPath: /app/superset_home
        livenessProbe:
          httpGet:
            path: /health
            port: 8088
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8088
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: superset-config
        configMap:
          name: superset-config

---
apiVersion: v1
kind: Service
metadata:
  name: superset-service
  namespace: analytics
spec:
  selector:
    app: superset
  ports:
  - port: 80
    targetPort: 8088
  type: LoadBalancer
```

### 3. Manual Installation dengan Pip

```bash
# Create virtual environment
python -m venv superset-env
source superset-env/bin/activate

# Install dependencies
pip install apache-superset
pip install psycopg2-binary
pip install redis

# Initialize database
superset db upgrade

# Create admin user
export FLASK_APP=superset
superset fab create-admin

# Load examples (optional)
superset load_examples

# Initialize Superset
superset init

# Start Superset
superset run -p 8088 --with-threads --reload --debugger
```

## 🔗 Database Configuration

### 1. Menghubungkan PostgreSQL Source Database

```python
# Database Connection String
postgresql://username:password@host:port/database_name
```

### 2. Configure Database di Superset

```bash
# Melalui UI atau CLI
superset set-database-uri \
    --database-name "Production PostgreSQL" \
    --uri postgresql://metabase_user:password@postgres-host:5432/your_database
```

### 3. Security Configuration

```sql
-- Create read-only user untuk Superset
CREATE USER superset_readonly WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE your_database TO superset_readonly;
GRANT USAGE ON SCHEMA mb TO superset_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA mb TO superset_readonly;

-- Enable untuk tabel future
ALTER DEFAULT PRIVILEGES IN SCHEMA mb 
GRANT SELECT ON TABLES TO superset_readonly;
```

## 📊 Dashboard Creation Workflow

### 1. Data Preparation & Modeling

#### a. Create Useful Views
```sql
-- View untuk sales summary
CREATE VIEW mb.vw_sales_dashboard AS
SELECT 
    DATE_TRUNC('month', tgl_invoice) as bulan,
    cabang,
    tipe_sparepart,
    COUNT(DISTINCT no_invoice) as total_invoice,
    SUM(total_harga) as total_penjualan,
    SUM(qty) as total_quantity,
    COUNT(DISTINCT nama_customer) as unique_customers,
    AVG(total_harga) as avg_transaction_value
FROM mb.data_transaksi_sparepart
GROUP BY DATE_TRUNC('month', tgl_invoice), cabang, tipe_sparepart;

-- View untuk product performance
CREATE VIEW mb.vw_product_performance AS
SELECT 
    kode_barang,
    nama_barang,
    tipe_sparepart,
    cabang,
    EXTRACT(YEAR FROM tgl_invoice) as tahun,
    EXTRACT(MONTH FROM tgl_invoice) as bulan,
    SUM(qty) as quantity_sold,
    SUM(total_harga) as revenue,
    COUNT(DISTINCT no_invoice) as transaction_count
FROM mb.data_transaksi_sparepart
GROUP BY kode_barang, nama_barang, tipe_sparepart, cabang, tahun, bulan;
```

#### b. Create Table Indexes
```sql
-- Optimize query performance
CREATE INDEX idx_transaksi_tgl_invoice ON mb.data_transaksi_sparepart(tgl_invoice);
CREATE INDEX idx_transaksi_cabang ON mb.data_transaksi_sparepart(cabang);
CREATE INDEX idx_transaksi_tipe_sparepart ON mb.data_transaksi_sparepart(tipe_sparepart);
CREATE INDEX idx_transaksi_kode_barang ON mb.data_transaksi_sparepart(kode_barang);
```

### 2. Creating Charts in Superset

#### Chart 1: Monthly Sales Trend
- **Chart Type**: Time Series Line Chart
- **Metrics**: 
  - SUM(total_penjualan) as "Total Penjualan"
- **Group By**: 
  - bulan (temporal)
- **Filters**:
  - cabang (optional)
  - tipe_sparepart (optional)

#### Chart 2: Branch Performance
- **Chart Type**: Bar Chart
- **Metrics**:
  - SUM(total_penjualan) as "Revenue"
  - COUNT(DISTINCT no_invoice) as "Total Transactions"
- **Group By**:
  - cabang

#### Chart 3: Product Category Distribution
- **Chart Type**: Pie Chart
- **Metrics**:
  - SUM(total_penjualan) as "Revenue"
- **Group By**:
  - tipe_sparepart

#### Chart 4: Customer Segmentation
- **Chart Type**: Big Number with Trend
- **Metrics**:
  - COUNT(DISTINCT nama_customer) as "Unique Customers"
- **Time Comparison**: Previous period

### 3. Building Interactive Dashboard

#### Dashboard: "Sales Performance Analytics"

**Layout Structure:**
```
┌─────────────────┬─────────────────┬─────────────────┐
│   Total Revenue │ Revenue Growth  │ Avg Order Value │
│   (Current MTD) │ (vs Last Month) │ (Current Month) │
├─────────────────┼─────────────────┼─────────────────┤
│  Monthly Sales  │   Revenue by    │  Top Performing │
│     Trend       │    Branch       │    Products     │
├─────────────────┼─────────────────┼─────────────────┤
│ Customer        │ Product Category│ Sales           │
│ Acquisition     │ Distribution    │ Forecast        │
└─────────────────┴─────────────────┴─────────────────┘
```

#### Dashboard Filters:
1. **Date Range** (tgl_invoice)
2. **Branch Selector** (cabang)
3. **Product Type** (tipe_sparepart)
4. **Customer Segment** (berdasarkan spending)

### 4. Advanced Chart Configurations

#### Custom SQL for Complex Metrics
```sql
-- Customer Retention Rate
WITH customer_months AS (
  SELECT 
    nama_customer,
    DATE_TRUNC('month', tgl_invoice) as bulan,
    LAG(DATE_TRUNC('month', tgl_invoice)) OVER (
      PARTITION BY nama_customer ORDER BY DATE_TRUNC('month', tgl_invoice)
    ) as prev_month
  FROM mb.data_transaksi_sparepart
  GROUP BY nama_customer, DATE_TRUNC('month', tgl_invoice)
)
SELECT
  bulan,
  COUNT(*) as active_customers,
  COUNT(CASE WHEN prev_month IS NOT NULL THEN 1 END) as retained_customers,
  ROUND(
    COUNT(CASE WHEN prev_month IS NOT NULL THEN 1 END) * 100.0 / COUNT(*),
    2
  ) as retention_rate
FROM customer_months
GROUP BY bulan
ORDER BY bulan;
```

## ⚡ Advanced Features

### 1. Caching Configuration
```python
# superset_config.py
FILTER_STATE_CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 86400,
    'CACHE_KEY_PREFIX': 'superset_filter_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0'
}

EXPLORE_FORM_DATA_CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 86400,
    'CACHE_KEY_PREFIX': 'superset_explore_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0'
}
```

### 2. Row Level Security
```python
# Define RLS rules
from superset import security_manager
from flask_appbuilder.security.sqla.models import Role

# Create RLS filter
rls_filter = {
    "clause": "cabang IN ('JAKARTA', 'BANDUNG')"
}

# Apply to specific role
role = security_manager.find_role("Alpha")
security_manager.add_permission_view_menu('datasource_access', 'your_table')
```

### 3. Custom Visualization Plugins
```javascript
// Custom chart plugin
export default class CustomSalesChart extends React.PureComponent {
  render() {
    const { data, height, width } = this.props;
    // Custom visualization logic
    return (
      <div>
        {/* Custom chart implementation */}
      </div>
    );
  }
}
```

### 4. API Integration
```python
# Python client untuk Superset API
import requests
import json

class SupersetClient:
    def __init__(self, base_url, username, password):
        self.base_url = base_url
        self.session = requests.Session()
        self.login(username, password)
    
    def login(self, username, password):
        login_data = {
            "username": username,
            "password": password,
            "provider": "db"
        }
        response = self.session.post(
            f"{self.base_url}/api/v1/security/login",
            json=login_data
        )
        self.session.headers.update({
            "Authorization": f"Bearer {response.json()['access_token']}"
        })
    
    def get_dashboard_data(self, dashboard_id):
        response = self.session.get(
            f"{self.base_url}/api/v1/dashboard/{dashboard_id}"
        )
        return response.json()
```

## 🏆 Best Practices

### 1. Performance Optimization
```sql
-- Use materialized views untuk heavy queries
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
    DATE(tgl_invoice) as tanggal,
    cabang,
    SUM(total_harga) as daily_revenue,
    COUNT(DISTINCT no_invoice) as daily_transactions
FROM mb.data_transaksi_sparepart
GROUP BY DATE(tgl_invoice), cabang;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW mv_daily_sales;
```

### 2. Dashboard Design Principles
- **Consistent Color Scheme**: Gunakan brand colors
- **Progressive Disclosure**: Tampilkan summary dulu, details on demand
- **Mobile Responsive**: Test di berbagai device sizes
- **Loading Optimization**: Gunakan cached queries untuk faster loading

### 3. Security Best Practices
```python
# superset_config.py
# Enable SSL/TLS
ENABLE_PROXY_FIX = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True

# CORS settings
ENABLE_CORS = True
CORS_OPTIONS = {
    'supports_credentials': True,
    'allow_headers': ['*'],
    'resources': ['*'],
    'origins': ['https://your-domain.com']
}
```

## 🔧 Maintenance & Monitoring

### 1. Health Checks
```bash
# Health check endpoint
curl http://localhost:8088/health

# API health check
curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:8088/api/v1/database/
```

### 2. Backup Strategy
```bash
#!/bin/bash
# backup_superset.sh

# Backup PostgreSQL database
pg_dump -h localhost -U superset superset > superset_backup_$(date +%Y%m%d).sql

# Backup configuration
tar -czf superset_config_$(date +%Y%m%d).tar.gz /app/superset_home

# Backup Redis data (jika perlu)
redis-cli SAVE
cp /var/lib/redis/dump.rdb redis_backup_$(date +%Y%m%d).rdb
```

### 3. Monitoring & Alerting
```yaml
# Prometheus metrics configuration
# superset_config.py
FEATURE_FLAGS = {
    'ALERT_REPORTS': True,
    'DASHBOARD_CACHE': True,
}

from superset.utils.log import DBEventLogger
class CustomEventLogger(DBEventLogger):
    def log(self, *args, **kwargs):
        # Custom logging logic
        super().log(*args, **kwargs)
```

### 4. Scaling Strategies
```yaml
# Horizontal scaling dengan multiple workers
superset worker --app celery
superset worker --app beat

# Gunicorn configuration
gunicorn \
    --bind 0.0.0.0:8088 \
    --workers 4 \
    --threads 8 \
    --timeout 120 \
    "superset.app:create_app()"
```

## 🚨 Troubleshooting

### Common Issues & Solutions

1. **Memory Issues**
```bash
# Increase memory limits
docker run -e SUPERSET_WORKERS=4 -e SUPERSET_WORKER_MEMORY=2GB ...
```

2. **Database Connection Pool**
```python
# superset_config.py
DATABASE_CONFIG = {
    'pool_pre_ping': True,
    'pool_recycle': 3600,
    'pool_size': 10,
    'max_overflow': 20
}
```

3. **Slow Query Performance**
```sql
-- Analyze query performance
EXPLAIN ANALYZE 
SELECT * FROM mb.data_transaksi_sparepart 
WHERE tgl_invoice >= '2024-01-01';
```

## 📈 Production Checklist

- [ ] Secret key di-set dengan strong value
- [ ] Database connection menggunakan SSL
- [ ] Row Level Security configured
- [ ] Caching mechanism enabled
- [ ] Backup strategy in place
- [ ] Monitoring dan alerting configured
- [ ] Load balancing setup (jika needed)
- [ ] Security headers configured
- [ ] Regular security updates

Dengan Apache Superset, Anda mendapatkan platform BI yang sangat powerful dan scalable untuk analisis data transaksi sparepart yang kompleks. Platform ini cocok untuk environment production dengan requirements enterprise-level.
