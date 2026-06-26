# Local MLOps Infrastructure: MLflow Tracking Server with MinIO & PostgreSQL

A production-grade, self-hosted Machine Learning Operations (MLOps) platform built natively on a Fedora workstation. This infrastructure establishes a persistent **MLflow Tracking Server** that stores structured metric data inside a local **PostgreSQL** instance and heavy binary model artifacts inside an on-premise **MinIO S3 Object Storage** bucket.

---

## 
Core Architecture Matrix
2. File Structural Setup
Create a dedicated working workspace folder to isolate background log structures, tracking components, and metadata footprints:
mkdir -p ~/mlflow_server
cd ~/mlflow_server

3. Dependency Installation
Install the primary Python workspace packages along with structural binary libraries to handle native PostgreSQL and AWS S3 api stream adapters:

pip install mlflow psycopg2-binary boto3

4. Create the Systemd Service
To ensure your MLflow server persists across system restarts and operates as an automated background application, establish a custom system configuration unit wrapper:

sudo nano /etc/systemd/system/mlflow.service


[Unit]
Description=MLflow Tracking Server Environment
After=network.target postgresql.service minio.service

[Service]
User=sethugopalan
WorkingDirectory=/home/sethugopalan/mlflow_server
Environment=AWS_ACCESS_KEY_ID=minioadmin
Environment=AWS_SECRET_ACCESS_KEY=your-secure-password-here123$
Environment=MLFLOW_S3_ENDPOINT_URL=http://localhost:9000
Environment=MLFLOW_S3_IGNORE_TLS=true
ExecStart=/home/sethugopalan/.local/bin/mlflow server \
    --backend-store-uri postgresql://postgres:welcome123$@localhost:5432/mlflow_backend \
    --default-artifact-root s3://mlflow-artifacts/ \
    --host 0.0.0.0 \
    --port 5000
Restart=always

[Install]
WantedBy=multi-user.target

5. Initialize the Tracking Daemon
Wake up your host system manager, scan for file changes, lock the profile on machine boot, and verify system operability:

# Refresh configurations
sudo systemctl daemon-reload

# Activate and boot background daemon
sudo systemctl enable --now mlflow

# Check operational diagnostics
sudo systemctl status mlflow

Network Exposure & Tunnel Routing
To expose the MLflow interactive interface dashboard (mlflow.terrafoxai.com) to authorized devices away from home, apply the following cluster configuration adjustments:

1. Ingress Rule Configuration
Append the web binding route rules inside your system configuration layout map located at /etc/cloudflared/config.yml (insert directly above the ultimate http_status: 404 catch-all line):

- hostname: mlflow.terrafoxai.com
    service: http://localhost:5000

2. Register DNS Upstream Edge Map
Execute this configuration utility parameter to pass the target domain profile safely into your Cloudflare records table

sudo cloudflared --config /etc/cloudflared/config.yml tunnel route dns <YOUR-TUNNEL-UUID-HERE> mlflow.terrafoxai.com

3. Deploy the Edge Stream Configuration
Restart your localized gateway connection utility manager to process the active adjustments:

sudo systemctl restart cloudflared


