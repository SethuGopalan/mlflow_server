# MLflow Server Setup on Fedora

This repository documents how to run a self-hosted MLflow Tracking Server for a private AI / data engineering lab.

The goal is:

```text
Coder projects / training scripts
        ↓
MLflow Tracking Server
        ↓
PostgreSQL = metadata backend
Object storage = artifact/model storage
```

This setup keeps MLflow as a shared tracking service instead of running a separate MLflow server inside every project.

---

## 1. Project Folder

Example local folder:

```bash
/home/<linux-user>/coder_workspace/mlflowserver
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

If the folder has permission issues, run from the host machine:

```bash
cd /home/<linux-user>/coder_workspace
sudo chown -R <linux-user>:<linux-user> mlflowserver
chmod -R 755 mlflowserver
```

Then create the basic structure:

```bash
cd /home/<linux-user>/coder_workspace/mlflowserver
mkdir -p logs scripts
touch README.md .env
```

---

## 2. Create Python Virtual Environment

Run this on the host machine:

```bash
cd /home/<linux-user>/coder_workspace/mlflowserver

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

## 4. Object Storage Artifact Bucket

MLflow uses artifact storage for files such as models, plots, datasets, and output files.

Create an artifact bucket in your S3-compatible object storage platform:

```text
mlflow-artifacts
```

Examples of S3-compatible object storage include MinIO or cloud object storage services.

Keep access keys and secret keys private.

---

## 5. Environment File

Open:

```bash
nano .env
```

Use this template and replace placeholders with your real local values:

```bash
# PostgreSQL backend store
MLFLOW_BACKEND_STORE_URI=postgresql://<postgres-user>:<postgres-password>@localhost:5432/mlflow_db

# Artifact store
MLFLOW_ARTIFACTS_DESTINATION=s3://mlflow-artifacts
MLFLOW_DEFAULT_ARTIFACT_ROOT=s3://mlflow-artifacts

# S3-compatible credentials
AWS_ACCESS_KEY_ID=<access-key>
AWS_SECRET_ACCESS_KEY=<secret-key>
MLFLOW_S3_ENDPOINT_URL=https://<object-storage-endpoint>
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
cd /home/<linux-user>/coder_workspace/mlflowserver
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

From the same host machine:

```text
http://localhost:5000
```

From another machine on the same private network:

```text
http://<mlflow-server-private-ip>:5000
```

If a secure public or private domain is configured through a reverse proxy or tunnel:

```text
https://<mlflow-domain>
```

Do not publish private IP addresses, real domains, access tokens, tunnel IDs, or credentials in a public repository.

---

## 8. Connect From a Python Project

In another project such as a dashboard or model training project:

```python
import mlflow

mlflow.set_tracking_uri("http://<mlflow-server-private-ip>:5000")
mlflow.set_experiment("example-project")

with mlflow.start_run():
    mlflow.log_param("model_type", "test")
    mlflow.log_metric("accuracy", 0.90)
```

If using a domain:

```python
import mlflow

mlflow.set_tracking_uri("https://<mlflow-domain>")
mlflow.set_experiment("example-project")
```

After running the script, check the MLflow UI.

---

## 9. Recommended Next Step

First confirm the manual command works.

After that, create a systemd service so MLflow starts automatically when the host machine restarts.

Example future service name:

```text
mlflow-server.service
```

---

## 10. Troubleshooting

### Permission denied inside the project folder

Run:

```bash
cd /home/<linux-user>/coder_workspace
sudo chown -R <linux-user>:<linux-user> mlflowserver
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

### Object storage artifact error

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

The host machine runs the shared MLflow Tracking Server. PostgreSQL stores MLflow metadata. S3-compatible object storage stores models and artifacts. Development workspaces, training scripts, and future compute nodes can all connect to the same MLflow tracking endpoint.
