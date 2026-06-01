# Jenkins + Gitea CI/CD Setup Guide

## Overview

```
Git push → Gitea webhook → Jenkins pipeline → Docker build → Nexus registry
```

### Infrastructure (docker-compose)

| Service  | URL                          | Purpose               |
|----------|------------------------------|-----------------------|
| Jenkins  | http://192.168.68.200:8080   | CI/CD pipelines       |
| Gitea    | http://192.168.68.200:8085   | Git repository        |
| Nexus    | http://192.168.68.200:8081   | Artifact/image repo   |
| Nexus Docker registry | http://192.168.68.200:8082 | Docker push/pull |

---

## Repo Structure

```
my-app/
├── app1/
│   ├── app1.py
│   ├── Dockerfile
│   └── requirements.txt
├── app2/
│   ├── app2.py
│   ├── Dockerfile
│   └── requirements.txt
├── k8s/
│   ├── app1.yaml
│   ├── app2.yaml
│   └── jaeger.yaml
└── README.md
```

Each app has its own `Dockerfile`. Jenkins builds each independently via separate pipelines.

---

## Step 1 — Nexus: Enable Docker Registry

### 1.1 Enable Docker Bearer Token Realm

1. Log into Nexus at `http://192.168.68.200:8081`
2. **Administration** → **Security** → **Realms**
3. Move **Docker Bearer Token Realm** from Available → Active
4. Save

### 1.2 Create Docker Hosted Repository

1. **Administration** → **Repositories** → **Create repository**
2. Choose **docker (hosted)**
3. Fill in:
   - Name: `docker-hosted`
   - HTTP port: `8082`
4. Save

---

## Step 2 — Gitea: Create Repo and Push Code

### 2.1 Create repository

1. Log into Gitea at `http://192.168.68.200:8085`
2. **+** → **New Repository** → name it `my-app` → **Create**

### 2.2 Push local code

```bash
git init
git add .
git commit -m "initial commit"
git remote add origin http://192.168.68.200:8085/<user>/my-app.git
git push -u origin master
```

---

## Step 3 — Host Docker: Allow Insecure Registry

Jenkins uses the host Docker socket. Tell Docker to trust the Nexus HTTP registry.

Edit `/etc/docker/daemon.json` on the **host**:

```json
{
  "insecure-registries": ["192.168.68.200:8082"]
}
```

Mount Docker socket and binary into Jenkins via docker-compose:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
  - /usr/bin/docker:/usr/bin/docker
```

Restart Docker on the host:

```bash
sudo systemctl restart docker
docker compose up -d jenkins
```

---

## Step 4 — Jenkins: Credentials

1. **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Fill in:
   - Kind: **Username with password**
   - Username: Nexus admin username
   - Password: Nexus admin password
   - ID: `nexus-docker-creds`
3. Save

---

## Step 5 — Jenkins: Install Plugins

**Manage Jenkins** → **Plugins** → **Available plugins**, install:

- **Generic Webhook Trigger**

Restart Jenkins after install.

---

## Step 6 — Jenkins: Create Pipelines

Create two **Pipeline** jobs: `app1-pipeline` and `app2-pipeline`.

### app1-pipeline

```groovy
pipeline {
    agent any

    triggers {
        GenericTrigger(
            causeString: 'Triggered by Gitea push',
            token: 'app1-token'
        )
    }

    environment {
        NEXUS_HOST  = "192.168.68.200:8082"
        IMAGE_NAME  = "app1"
        IMAGE_TAG   = "${BUILD_NUMBER}"
        FULL_IMAGE  = "${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/master']],
                    userRemoteConfigs: [[
                        url: 'http://192.168.68.200:8085/<user>/my-app.git'
                    ]]
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${FULL_IMAGE} ./app1"
            }
        }

        stage('Push to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo "${NEXUS_PASS}" | docker login ${NEXUS_HOST} -u ${NEXUS_USER} --password-stdin
                        docker push ${FULL_IMAGE}
                        docker logout ${NEXUS_HOST}
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${FULL_IMAGE} || true"
        }
    }
}
```

### app2-pipeline

Same as above with two changes:
- `IMAGE_NAME = "app2"`
- `docker build -t ${FULL_IMAGE} ./app2`
- `token: 'app2-token'`

---

## Step 7 — Gitea: Allow Webhook Outbound Calls

By default Gitea blocks webhooks to local IPs. Add this to the `gitea` service in docker-compose:

```yaml
environment:
  - GITEA__webhook__ALLOWED_HOST_LIST=192.168.68.200
```

Restart Gitea:

```bash
docker compose restart gitea
```

---

## Step 8 — Gitea: Add Webhooks

Go to `http://192.168.68.200:8085/<user>/my-app` → **Settings** → **Webhooks** → **Add Webhook** → **Gitea**

Add two webhooks:

| Field        | Webhook 1 (app1)                                                                 | Webhook 2 (app2)                                                                 |
|--------------|------------------|------------------|
| Target URL   | `http://192.168.68.200:8080/generic-webhook-trigger/invoke?token=app1-token`    | `http://192.168.68.200:8080/generic-webhook-trigger/invoke?token=app2-token`    |
| Content Type | `application/json`                                                               | `application/json`                                                               |
| Trigger      | Push events                                                                      | Push events                                                                      |

Click **Test Delivery** — both should return HTTP 200.

---

## Testing

```bash
echo "test" >> README.md
git add . && git commit -m "test webhook trigger"
git push origin master
```

Both `app1-pipeline` and `app2-pipeline` should trigger automatically within seconds.

---

## Verifying Images in Nexus

1. Go to `http://192.168.68.200:8081`
2. **Browse** → `docker-hosted`
3. You should see `app1` and `app2` with build number tags

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `docker push` fails with HTTP error | Missing insecure-registry | Step 3 |
| `docker: not found` in Jenkins | Binary not mounted | Mount `/usr/bin/docker` in compose |
| Webhook returns connection refused | Gitea blocking local IPs | Step 7 |
| `GenericTrigger` compilation error | Plugin not installed | Step 5 |
| Pipeline doesn't trigger on push | Webhook not configured | Step 8 |
| Nexus login returns 401 | Docker Bearer Token Realm not active | Step 1.1 |
