# MLflow Server Setup on Fedora

This folder is used to manage a Fedora-hosted MLflow Tracking Server for the private AI lab.

The goal is:

```text
Coder projects / training scripts
        ↓
MLflow Tracking Server on Fedora
        ↓
PostgreSQL = metadata backend
MinIO = artifact/model storage
```

This setup keeps MLflow as a shared service instead of running a separate MLflow server inside every project.

---

## 1. Project Folder

Main folder:

```bash
/home/sethugopalan/coder_workspace/mlflowserver
```

Expected structure:

```text
mlflowserver/
  .env
  .venv/
  README.md
  logs/
  scripts/
```

If the folder has permission issues, run from Fedora:

```bash
cd /home/sethugopalan/coder_workspace
sudo chown -R sethugopalan:sethugopalan mlflowserver
chmod -R 755 mlflowserver
```

Then create the basic structure:

```bash
cd /home/sethugopalan/coder_workspace/mlflowserver
mkdir -p logs scripts
touch README.md .env
```

---

## 2. Create Python Virtual Environment

Run this on Fedora:

```bash
cd /home/sethugopalan/coder_workspace/mlflowserver

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install mlflow psycopg2-binary boto3 python-dotenv
```

Check MLflow:

```bash
mlflow --version
```

---

## 3. PostgreSQL Backend Database

MLflow uses the backend database to store experiment metadata such as runs, parameters, metrics, tags, and model registry metadata.

Create a database for MLflow:

```bash
sudo -u postgres psql
```

Inside PostgreSQL:

```sql
CREATE DATABASE mlflow_db;
\q
```

If you already have a PostgreSQL user/password, use that user in the `.env` file.

---

## 4. MinIO Artifact Bucket

MLflow uses artifact storage for files such as models, plots, datasets, and output files.

In MinIO Console, create this bucket:

```text
mlflow-artifacts
```

Keep your MinIO access key and secret key private.

---

## 5. Environment File

Open:

```bash
nano .env
```

Use this template and replace placeholders with your real local values:

```bash
# PostgreSQL backend store
MLFLOW_BACKEND_STORE_URI=postgresql://POSTGRES_USER:POSTGRES_PASSWORD@localhost:5432/mlflow_db

# MinIO artifact store
MLFLOW_ARTIFACTS_DESTINATION=s3://mlflow-artifacts
MLFLOW_DEFAULT_ARTIFACT_ROOT=s3://mlflow-artifacts

# MinIO S3-compatible credentials
AWS_ACCESS_KEY_ID=YOUR_MINIO_ACCESS_KEY
AWS_SECRET_ACCESS_KEY=YOUR_MINIO_SECRET_KEY
MLFLOW_S3_ENDPOINT_URL=https://minio.terrafoxai.com
```

Important:

```text
Do not push .env to GitHub.
```

Add this to `.gitignore`:

```gitignore
.env
.venv/
logs/
__pycache__/
*.pyc
```

---

## 6. Start MLflow Server Manually

Run:

```bash
cd /home/sethugopalan/coder_workspace/mlflowserver
source .venv/bin/activate

set -a
source .env
set +a

mlflow server \
  --backend-store-uri "$MLFLOW_BACKEND_STORE_URI" \
  --artifacts-destination "$MLFLOW_ARTIFACTS_DESTINATION" \
  --host 0.0.0.0 \
  --port 5000
```

If your MLflow version does not accept `--artifacts-destination`, use this fallback:

```bash
mlflow server \
  --backend-store-uri "$MLFLOW_BACKEND_STORE_URI" \
  --default-artifact-root "$MLFLOW_DEFAULT_ARTIFACT_ROOT" \
  --host 0.0.0.0 \
  --port 5000
```

---

## 7. Open MLflow UI

From the Fedora browser or another machine on the same network:

```text
http://192.168.1.181:5000
```

Later, after Cloudflare Tunnel is configured:

```text
https://mlflow.terrafoxai.com
```

---

## 8. Connect From a Python Project

In another project such as the diabetes dashboard or model training project:

```python
import mlflow

mlflow.set_tracking_uri("http://192.168.1.181:5000")
mlflow.set_experiment("diabetes-dashboard")

with mlflow.start_run():
    mlflow.log_param("model_type", "test")
    mlflow.log_metric("accuracy", 0.90)
```

After running the script, check the MLflow UI.

---

## 9. Recommended Next Step

First confirm the manual command works.

After that, create a systemd service so MLflow starts automatically when Fedora restarts.

Example future service name:

```text
mlflow-server.service
```

---

## 10. Troubleshooting

### Permission denied inside mlflowserver folder

Run:

```bash
cd /home/sethugopalan/coder_workspace
sudo chown -R sethugopalan:sethugopalan mlflowserver
chmod -R 755 mlflowserver
```

### PostgreSQL connection failed

Check PostgreSQL status:

```bash
sudo systemctl status postgresql
```

Check database exists:

```bash
sudo -u postgres psql -l
```

### MinIO artifact error

Check:

```bash
echo $MLFLOW_S3_ENDPOINT_URL
echo $AWS_ACCESS_KEY_ID
```

Do not print or share the secret key publicly.

### Port 5000 already in use

Run:

```bash
sudo lsof -i :5000
```

Stop the old process or use a different port.

---

## Architecture Note

This is a private AI lab setup.

Fedora hosts the shared MLflow Tracking Server. PostgreSQL stores MLflow metadata. MinIO stores models and artifacts. Coder projects, training scripts, and future Jetson workloads can all connect to the same MLflow tracking endpoint.
