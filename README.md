# Capstone Project — Production-ready ML Service (Flask + MLOps)

> **Short pitch:** A full end-to-end machine learning project showing best practices for data engineering, experimentation (MLflow/Dagshub), reproducible pipelines (DVC), containerization (Docker), CI/CD, and cloud deployment (EKS). Designed to be demonstrable, production-friendly, and easy to follow for reviewers and hiring managers.

---

## Table of contents

1. [Project Overview](#project-overview)
2. [Key Features](#key-features)
3. [Architecture at-a-glance](#architecture-at-a-glance)
4. [Getting started (quick)](#getting-started-quick)
5. [Detailed setup & commands](#detailed-setup--commands)

   * Project structure
   * Local environment & cookiecutter
   * Dagshub / MLflow
   * DVC & remote storage
   * Flask app
   * Docker & local container run
   * CI/CD and GitHub Actions
   * EKS deployment
6. [Monitoring: Prometheus & Grafana](#monitoring-prometheus--grafana)
7. [Cleanup & rollback](#cleanup--rollback)
8. [Notes, tips & troubleshooting](#notes-tips--troubleshooting)
9. [Contributing](#contributing)
10. [License](#license)

---

## Project Overview

This repository demonstrates how to build and operate a machine learning web service using modern MLOps practices. It contains

* reproducible data pipelines (DVC + `params.yaml` + `dvc.yaml`),
* experiment tracking with MLflow integrated to Dagshub,
* model training, evaluation and registration,
* a Flask REST API that serves the model,
* containerization (Docker) and CI/CD that builds and pushes to ECR,
* Kubernetes deployment on EKS, and
* production monitoring using Prometheus and Grafana.

This README documents a clean, repeatable setup so anyone (engineer, hiring manager, or reviewer) can run and evaluate the project.

---

## Key Features

* Reproducible pipeline: `dvc repro` to run preprocessing → training → evaluation.
* Experiment tracking: MLflow runs viewable on Dagshub.
* CI/CD: GitHub Actions builds and pushes Docker image to ECR (configurable).
* Cloud-ready: EKS deployment with an external LoadBalancer for the Flask app.
* Observability: Prometheus scrapes app metrics, Grafana dashboards visualize them.

---

## Architecture at-a-glance

```
Local dev  -->  GitHub  -->  Dagshub (MLflow)      
   |            |            |                       
   |            v            v                       
   |        CI/CD (build) -> ECR -> EKS  -> LB -> App
   |                                    \          
   |                                     -> Prometheus -> Grafana
   v
 DVC (data) -> S3 (remote storage)
```

---

## Getting started (quick)

1. Clone the repo

```bash
git clone <your-repo-url>
cd <repo-directory>
```

2. Create & activate conda env

```bash
conda create -n atlas python=3.10 -y
conda activate atlas
```

3. Install dependencies

```bash
pip install -r requirements.txt
```

4. Initialize DVC & run pipeline (local test)

```bash
dvc init
dvc repro
```

5. Build Docker image and run locally

```bash
docker build -t capstone-app:latest .
docker run -p 8888:5000 -e CAPSTONE_TEST=<token-or-secret> capstone-app:latest
```

---

## Detailed setup & commands

### 1) Project structure (recommended)

```
├── data/                 # raw/processed (DVC tracked)
├── flask_app/            # Flask service + API endpoints
├── notebooks/            # experiments & EDA
├── src/                  # project code
│   ├── logger.py
│   ├── data_ingestion.py
│   ├── data_preprocessing.py
│   ├── feature_engineering.py
│   ├── model_building.py
│   ├── model_evaluation.py
│   └── register_model.py
├── dvc.yaml
├── params.yaml
├── requirements.txt
├── Dockerfile
├── .github/workflows/ci.yaml
└── README.md
```

> Note: `src.models` was renamed to `src.model` for consistency in imports.

### 2) Bootstrap using cookiecutter

```bash
pip install cookiecutter
cookiecutter -c v1 https://github.com/drivendata/cookiecutter-data-science
# then adapt generated structure to this repo
```

### 3) Dagshub + MLflow

1. Create repository on Dagshub and connect to your GitHub repo (Repository → Connect).
2. Grab MLflow experiment tracking URL from the Dagshub repo settings and configure `MLFLOW_TRACKING_URI` in your environment or code.
3. Install packages:

```bash
pip install dagshub mlflow
```

4. Use MLflow API in `notebooks/` and `src/` to log params, metrics, and artifacts.

**Tip:** Keep tokens and secrets in GitHub Secrets or environment variables — never commit them.

### 4) DVC and remote storage

```bash
dvc init
mkdir local_s3
# local testing only
dvc remote add -d mylocal local_s3
# create params.yaml and dvc.yaml (pipelines up to model evaluation)
dvc repro
# after testing, add real S3 remote
pip install 'dvc[s3]' awscli
aws configure
dvc remote add -d myremote s3://<bucket-name>
dvc push
```

Commands to inspect/remove remotes:

```bash
dvc remote list
dvc remote remove mylocal
```

### 5) Flask app

* Location: `flask_app/`
* Run locally:

```bash
cd flask_app
pip install -r ../requirements.txt
python app.py  # or run the provided start script
```

Expose metrics (Prometheus client) on an endpoint (e.g., `/metrics`) so Prometheus can scrape it.

### 6) Docker

* Generate a minimal `requirements.txt` (we use `pipreqs`):

```bash
pip install pipreqs
cd flask_app
pipreqs . --force
```

* Build & run image

```bash
# from repo root
docker build -t capstone-app:latest .
# run (set required env vars)
docker run -p 8888:5000 -e CAPSTONE_TEST=<your-value> capstone-app:latest
```

> If you see `OSError: CAPSTONE_TEST environment variable is not set`, pass the env var as shown above.

### 7) CI/CD (GitHub Actions)

* Add `.github/workflows/ci.yaml` to:

  * install deps
  * run tests
  * build Docker image
  * login to ECR
  * push image to ECR

**Secrets to store in GitHub repository settings:**

* `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_ACCOUNT_ID`, `ECR_REPOSITORY` (name)

> Generate tokens (Dagshub/GitHub) and save them as secrets. Never commit secrets in code.

### 8) EKS deployment checklist

1. Install/verify tools: `aws`, `kubectl`, `eksctl`.
2. Create cluster (example):

```bash
eksctl create cluster --name flask-app-cluster --region us-east-1 \
  --nodegroup-name flask-app-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 1 --managed
```

3. Update kubeconfig:

```bash
aws eks --region us-east-1 update-kubeconfig --name flask-app-cluster
```

4. Deploy resources: `kubectl apply -f deployment.yaml` and `kubectl apply -f service.yaml`.
5. Verify:

```bash
kubectl get nodes
kubectl get namespaces
kubectl get pods
kubectl get svc
```

6. When LoadBalancer is ready, fetch external IP and test:

```bash
kubectl get svc flask-app-service
curl http://<external-ip>:5000
```

**Notes:** adjust security group inbound rule for port `5000`.

## Monitoring: Prometheus & Grafana

**Prometheus setup (on an EC2)**

* Install Prometheus, add your app as a scrape target in `/etc/prometheus/prometheus.yml`.
* Example `scrape_configs` entry should point to your LB DNS and `/metrics` port.

**Grafana setup (on an EC2)**

* Install Grafana, start the service and open `http://<ec2-ip>:3000`.
* Add Prometheus as a data source and build dashboards.

## Cleanup & rollback

Common cleanup commands:

```bash
kubectl delete deployment flask-app
kubectl delete service flask-app-service
kubectl delete secret capstone-secret
eksctl delete cluster --name flask-app-cluster --region us-east-1
# ECR & S3 can be deleted from AWS console
```

Also verify CloudFormation stacks and confirm termination if needed.

## Notes, tips & troubleshooting

* If `aws` CLI conflicts with an Anaconda-installed version, uninstall the Python package `awscli` and install AWS CLI MSI for Windows. Then ensure the `.exe` path (e.g. `C:\Program Files\Amazon\AWSCLIV2\`) is in `PATH`.
* For Windows, download `kubectl.exe` and `eksctl.exe` and move to a PATH directory such as `C:\Windows\System32`.
* Fleet Requests quota errors when creating NodeGroups are caused by account limits on EC2 Fleet Requests — request quota increase or clean up existing ASGs.
* Always use environment variables or secrets storage for tokens — never commit them.

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feat/your-feature`
3. Make changes & add tests
4. Run CI locally if possible
5. Open a PR with a clear description and link to MLflow runs if relevant

## License

MIT © Your Name

---



<!-- End of README -->
