##1
## My Manual Setup Steps (Ubuntu)

### 1) Clone the repo
```bash
git clone https://github.com/harikayarabala/lms-latest.git
cd lms-latest

##2) Backend setup (API)
cd api
npm install

##Create .env in api/:
DATABASE_URL="postgresql://lms_user:lms_pass@localhost:5432/lms_db"
PORT=5000
NODE_ENV=development

##Prisma:
npx prisma generate
npx prisma migrate dev

##Run backend:
npm run dev

##verify:
curl http://localhost:5000/api

##3) Database setup (PostgreSQL 16)
psql --version

##Create user & DB:

sudo -u postgres psql
CREATE USER lms_user WITH PASSWORD 'lms_pass';
CREATE DATABASE lms_db OWNER lms_user;
GRANT ALL PRIVILEGES ON DATABASE lms_db TO lms_user;
ALTER USER lms_user CREATEDB;
\q

##Verify connection:

psql "postgresql://lms_user:lms_pass@localhost:5432/lms_db"

##4) Frontend setup (Webapp)
cd ../webapp
npm install


##Update webapp/.env:

VITE_API_URL=http://localhost:5000/api


##Run frontend:

npm run dev


##Open:

http://localhost:3000

##5) Verify DB data (UI → DB)

##Create a course from the UI (example: “Devops”), then verify in DB:

psql "postgresql://lms_user:lms_pass@localhost:5432/lms_db" -c 'SELECT * FROM "Course";'

##Ports

Frontend: http://localhost:3000

Backend: http://localhost:5000

DB: localhost:5432


UI → Backend → Database Flow Verification

Open the UI in browser:

http://localhost:3000


##Login as Admin (or use available admin access).

Navigate to Courses section.

Click Add Course and create a course
Example:

##Course Name: Devops


#Submit the form.

##Verify Data Stored in Database

##After creating the course from UI, verify data in PostgreSQL:

psql "postgresql://lms_user:lms_pass@localhost:5432/lms_db" \
-c 'SELECT * FROM "Course";'


##This confirms:

UI → Backend API → PostgreSQL DB


data flow is working correctly.




LMS Project – Dockerized Setup (Without docker-compose)

This project is a Learning Management System (LMS) consisting of three independent services, each running in its own Docker container without using docker-compose.

🧩 Architecture Overview
Browser
   |
   |--> Frontend (React + Nginx)  →  http://localhost:3000
             |
             |--> Backend API (Node.js + Prisma) → http://localhost:5000
                           |
                           |--> PostgreSQL DB → lms-db:5432

📂 Project Structure
lms-project/
├── api/        # Backend (Node.js, Express, Prisma)
├── webapp/     # Frontend (React, built & served via Nginx)
└── README.md   # Documentation

🛠 Prerequisites

Docker installed and running

Git installed

Verify:

docker --version
git --version

🚀 Setup & Run (Step by Step)
1️⃣ Clone Repository
git clone https://github.com/harikayarabala/lms-project.git
cd lms-project

2️⃣ Create Docker Network

This allows containers to communicate using container names.

docker network create lms-net

🗄 Database Setup (PostgreSQL)
3️⃣ Run PostgreSQL Container
docker run -d \
  --name lms-db \
  --network lms-net \
  -e POSTGRES_USER=lms_user \
  -e POSTGRES_PASSWORD=lms_pass \
  -e POSTGRES_DB=lms_db \
  -p 5433:5432 \
  -v lms-db-data:/var/lib/postgresql/data \
  postgres:16


Details:

DB name: lms_db

Username: lms_user

Password: lms_pass

Host access: localhost:5433

Docker access: lms-db:5432

🔧 Backend Setup (API)
4️⃣ Build Backend Image
cd api
docker build -t lms-api:1.0 .

5️⃣ Run Backend Container
docker run -d \
  --name lms-api \
  --network lms-net \
  -p 5000:5000 \
  -e DATABASE_URL="postgresql://lms_user:lms_pass@lms-db:5432/lms_db" \
  -e NODE_ENV=production \
  -e PORT=5000 \
  lms-api:1.0

6️⃣ Verify Backend
curl http://localhost:5000/api


Expected response:

{"message":"API is running"}


Check logs:

docker logs -f lms-api

🌐 Frontend Setup (WebApp)
7️⃣ Build Frontend Image
cd ../webapp
docker build -t lms-web:1.0 .

8️⃣ Run Frontend Container
docker run -d \
  --name lms-web \
  --network lms-net \
  -p 3000:80 \
  lms-web:1.0

✅ Access Application

Frontend UI: http://localhost:3000

Backend API: http://localhost:5000/api

🧪 Verify Data in Database
List tables
docker exec -it lms-db psql -U lms_user -d lms_db -c '\dt'

View courses

⚠️ Table names are case-sensitive

docker exec -it lms-db psql -U lms_user -d lms_db \
  -c 'SELECT * FROM "Course" ORDER BY id DESC;'

🧹 Cleanup Commands
Stop containers
docker stop lms-web lms-api lms-db

Remove containers
docker rm lms-web lms-api lms-db

Remove database volume (⚠️ deletes data)
docker volume rm lms-db-data

Remove network
docker network rm lms-net

📝 Key Notes

❌ No docker-compose is used

✅ Each service is managed independently

✅ Containers communicate using Docker network (lms-net)

✅ Backend connects to DB using container name lms-db

**Deploying the application using the docker :**

pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'GIT_URL', defaultValue: 'https://github.com/harikayarabala/lms-project.git', description: 'Git repo URL')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')

    // Scans / deploy switches
    booleanParam(name: 'RUN_TRIVY', defaultValue: false, description: 'Run Trivy image scan (requires trivy on agent)')
    booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Deploy using docker compose up -d')

    // Control behavior
    booleanParam(name: 'FAIL_ON_TEST', defaultValue: false, description: 'If true, pipeline fails when tests fail')
    booleanParam(name: 'FAIL_ON_QUALITY_GATE', defaultValue: false, description: 'If true, pipeline fails when SonarQube Quality Gate fails')
  }

  environment {
    // Must match Jenkins -> Manage Jenkins -> System -> SonarQube servers -> Name
    SONARQUBE_SERVER_NAME = "sonarqube"

    // Must match Jenkins -> Global Tool Configuration -> SonarQube Scanner -> Name
    SONAR_SCANNER_TOOL = "sonar-scanner"

    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  tools {
    // Must match Jenkins -> Global Tool Configuration -> NodeJS -> Name
    nodejs "node20"
  }

  stages {

    stage('Checkout (manual)') {
      steps {
        cleanWs()
        git branch: params.GIT_BRANCH, url: params.GIT_URL
        sh(label: 'versions', script: 'node -v; npm -v; docker version >/dev/null 2>&1 || true; docker compose version || true')
        sh(label: 'list workspace', script: 'ls -la')
      }
    }

    stage('Install Dependencies') {
      steps {
        sh(label: 'npm ci (api + webapp)', script: '''
          bash -lc '
            set -euo pipefail

            if [ -d "api" ]; then
              cd api
              npm ci
              cd ..
            fi

            if [ -d "webapp" ]; then
              cd webapp
              npm ci
              cd ..
            fi
          '
        ''')
      }
    }

    stage('Build') {
      steps {
        sh(label: 'npm run build', script: '''
          bash -lc '
            set -euo pipefail

            if [ -d "api" ]; then
              cd api
              npm run build --if-present
              cd ..
            fi

            if [ -d "webapp" ]; then
              cd webapp
              npm run build --if-present
              cd ..
            fi
          '
        ''')
      }
    }

    stage('Test') {
      steps {
        script {
          // Run tests but by default do NOT fail the whole pipeline if tests fail.
          def testStatus = sh(
            label: 'npm test (api + webapp)',
            returnStatus: true,
            script: '''
              bash -lc '
                set -euo pipefail

                status=0

                if [ -d "api" ]; then
                  cd api
                  npm test --if-present || status=$?
                  cd ..
                fi

                if [ -d "webapp" ]; then
                  cd webapp
                  npm test --if-present || status=$?
                  cd ..
                fi

                exit $status
              '
            '''
          )

          if (testStatus != 0) {
            echo "⚠️ Tests returned non-zero exit code: ${testStatus}"
            if (params.FAIL_ON_TEST) {
              error("Failing pipeline because FAIL_ON_TEST=true")
            } else {
              echo "Continuing pipeline (FAIL_ON_TEST=false)"
            }
          } else {
            echo "✅ Tests passed"
          }
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        script {
          def scannerHome = tool(env.SONAR_SCANNER_TOOL)
          withSonarQubeEnv(env.SONARQUBE_SERVER_NAME) {
            sh(label: 'sonar-scanner', script: """
              bash -lc '
                set -euo pipefail
                ${scannerHome}/bin/sonar-scanner
              '
            """)
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          // If FAIL_ON_QUALITY_GATE=true, abortPipeline will fail the build when gate is RED.
          // If false, pipeline continues even if gate fails (useful for demo/learning).
          def abortOnGate = params.FAIL_ON_QUALITY_GATE

          timeout(time: 20, unit: 'MINUTES') {
            def qg = waitForQualityGate abortPipeline: abortOnGate
            echo "Quality Gate Status: ${qg.status}"
          }

          if (params.FAIL_ON_QUALITY_GATE == false) {
            echo "Continuing even if Quality Gate is not OK (FAIL_ON_QUALITY_GATE=false)."
          }
        }
      }
    }

    stage('Docker Build (compose)') {
      steps {
        sh(label: 'docker compose build', script: '''
          bash -lc '
            set -euo pipefail

            test -f docker-compose.yml

            docker version
            docker compose version

            docker compose build --no-cache
            docker compose config
          '
        ''')
      }
    }

    stage('Image Scan (Trivy - optional)') {
      when { expression { return params.RUN_TRIVY } }
      steps {
        sh(label: 'trivy scan', script: '''
          bash -lc '
            set -euo pipefail

            command -v trivy >/dev/null 2>&1 || {
              echo "Trivy not installed on agent. Disable RUN_TRIVY or install trivy."
              exit 1
            }

            echo "Recent images:"
            docker images --format "{{.Repository}}:{{.Tag}}" | head -n 15
          '
        ''')
      }
    }

    stage('Deploy (docker compose up -d)') {
      when { expression { return params.DEPLOY } }
      steps {
        sh(label: 'docker compose up', script: '''
          bash -lc '
            set -euo pipefail

            docker compose down || true
            docker compose up -d
            docker compose ps
          '
        ''')
      }
    }
  }

  post {
    always {
      sh(label: 'compose ps (always)', script: '''
        bash -lc '
          set +e
          docker compose ps || true
        '
      ''')
    }
    success {
      echo "✅ Pipeline completed successfully."
    }
    failure {
      echo "❌ Pipeline failed. Check stage logs above."
    }
  }
}





LMS Project – Manual Kubernetes Deployment Guide
📌 Project Architecture
Frontend (lms-web)
        ↓
Nginx Reverse Proxy (/api)
        ↓
Backend (lms-api)
        ↓
PostgreSQL Database

🧱 1️⃣ Prerequisites

Ensure the following are installed:

Docker

kubectl

kind (Kubernetes in Docker)

Git

Check versions:

docker --version
kubectl version --client
kind --version

🏗 2️⃣ Create Kubernetes Cluster (Kind)
kind create cluster
kubectl get nodes


Expected:

kind-control-plane   Ready

📁 3️⃣ Create Namespace
kubectl create namespace lms
kubectl get ns


Namespace isolates project resources.

🐘 4️⃣ Deploy PostgreSQL (Database Layer)

Apply manifest:

kubectl apply -f postgres.yaml


Verify:

kubectl -n lms get pods,svc


Expected:

Postgres pod running

ClusterIP service on port 5432

Database connection string used by backend:

postgresql://postgres:itjustworks@postgres.lms.svc.cluster.local:5432/mydb

⚙️ 5️⃣ Deploy Backend (lms-api)

Apply manifest:

kubectl apply -f lms-api.yaml


Verify:

kubectl -n lms get pods,svc


Test backend using port-forward:

kubectl -n lms port-forward svc/lms-api 3000:3000


In another terminal:

curl http://127.0.0.1:3000/api


Expected:

{"message":"success","env":"production"}

🌐 6️⃣ Deploy Frontend (lms-web)

Build image:

docker build -t lms-web:1.1 ./webapp


Load image into kind:

kind load docker-image lms-web:1.1


Apply manifest:

kubectl apply -f lms-web.yaml


Verify:

kubectl -n lms get pods

🔁 7️⃣ Configure Nginx Reverse Proxy

Inside webapp/nginx.conf:

location /api/ {
    proxy_pass http://lms-api.lms.svc.cluster.local:3000/api/;
}


Rebuild image after changes:

docker build -t lms-web:1.1 ./webapp
kind load docker-image lms-web:1.1
kubectl -n lms rollout restart deployment lms-web

🧪 8️⃣ Access Application

Port forward frontend:

kubectl -n lms port-forward svc/lms-web 3001:80


Open:

http://127.0.0.1:3001


Test API through frontend:

curl http://127.0.0.1:3001/api/

🔎 9️⃣ Verification Commands

Check pods:

kubectl -n lms get pods


Check logs:

kubectl -n lms logs <pod-name>


Check rollout status:

kubectl -n lms rollout status deployment/lms-web

🧹 🔁 Clean Up

Delete namespace:

kubectl delete namespace lms


Delete cluster:

kind delete cluster

📦 YAML Files Used

postgres.yaml

lms-api.yaml

lms-web.yaml

🏆 What This Deployment Demonstrates

✅ Namespace isolation
✅ Multi-tier architecture
✅ Internal service communication
✅ ClusterIP services
✅ Nginx reverse proxy
✅ Environment variable configuration
✅ Port-forward testing
✅ Image loading into kind
✅ Rolling updates

🎯 Interview Explanation (Short Version)

I deployed a 3-tier application manually in Kubernetes using namespace isolation.
PostgreSQL runs as a ClusterIP service, backend connects using internal DNS, and frontend uses Nginx reverse proxy to route /api to backend service.
I validated using port-forward and rollout status.


Kubernetes Automation Documentation (Jenkins + Docker + KIND)
1) Overview

This project uses Jenkins Pipeline to automate:

✅ Code checkout from GitHub

✅ Install dependencies (Node/NPM)

✅ Build API + Webapp

✅ Run unit tests (Vitest)

✅ SonarQube code scan + Quality Gate

✅ Docker image build (API + Web)

✅ Load images into KIND cluster

✅ Deploy manifests to Kubernetes

✅ Print application URL (NodePort)

2) Prerequisites
Jenkins Server (Agent/Node) must have:

Jenkins (Pipeline plugin enabled)

Docker

kubectl

kind

NodeJS (configured in Jenkins tools)

SonarQube Scanner (installed OR container-based run)

Access to Kubernetes context (kind-kind)

Check versions:

docker --version
kubectl version --client=true
kind version
node -v
npm -v

3) Required Jenkins Plugins

Install these plugins:

Pipeline

Git

SonarQube Scanner for Jenkins

Quality Gates (SonarQube)

Credentials Binding

4) Jenkins Credentials Setup
A) SonarQube Token

Go to: Jenkins → Manage Jenkins → Credentials

Add:

Kind: Secret text

ID: sonarqube-token

Secret: (paste your SonarQube token)

B) SonarQube Server Config

Jenkins → Manage Jenkins → Configure System

SonarQube servers:

Name: sonarqube

Server URL: http://172.30.97.180:9001

Authentication token: select sonarqube-token

5) Kubernetes Cluster Setup (KIND)
Verify cluster exists:
kind get clusters
kubectl config get-contexts
kubectl config current-context
kubectl get nodes


Expected:

context: kind-kind

node: kind-control-plane (Ready)

6) Kubernetes Namespace & Resources

Namespace used:

lms

Verify:

kubectl -n lms get pods -o wide
kubectl -n lms get svc


Expected services:

api-server (ClusterIP 5000)

web-server (NodePort 30080)

7) Pipeline Flow (What Each Stage Does)
Stage 1: Checkout

Pulls latest code from GitHub

Stage 2: Install Dependencies

Runs:

npm ci in api/

npm ci in webapp/

Stage 3: Build

Runs:

API: npm run build

Web: npm run build

Stage 4: Test

Runs:

Web tests: npm test
(If FAIL_ON_TEST=false, pipeline continues even if tests fail)

Stage 5: SonarQube Scan

Runs:

sonar-scanner

Stage 6: Quality Gate

Waits for SonarQube result:

PASS → continue

FAIL → pipeline fails (optional)

Stage 7: Docker Build + Load into KIND

Build images:

docker build -t lms-public-api:$TAG api/

docker build -t lms-web:$TAG webapp/

Load into kind:

kind load docker-image ... --name kind

Stage 8: Deploy to Kubernetes

Applies YAML manifests:

postgres.yaml

lms-api.yaml

lms-web.yaml

Rollout check:

kubectl rollout status ...

Stage 9: Print App URL

Fetch node IP and print:

http://<node-ip>:30080

8) Accessing the Application
Option A (NodePort)
kubectl -n lms get svc web-server


Open:

http://<NODE_IP>:30080

Option B (Port Forward)

If NodePort not reachable:

kubectl -n lms port-forward svc/web-server 8090:80 --address 0.0.0.0


Open:

http://<NODE_IP>:8090

9) Common Issues & Fixes
Issue: error: no context exists with the name kind-kind

Fix:

sudo -u jenkins kubectl config get-contexts
sudo -u jenkins kubectl config use-context kind-kind

Issue: Node/NPM not found in Jenkins

Fix:

Manage Jenkins → Global Tool Configuration → NodeJS

Use tool('node20') in Jenkinsfile

Issue: Webapp API calls failing (504 upstream timeout)

Fix:

Ensure Nginx proxy passes to correct service:

/api/ → api-server:5000 OR correct API service

Issue: Port already in use during port-forward

Fix:

sudo lsof -i :8080
sudo kill -9 <PID>


Or use different port:

kubectl port-forward svc/web-server 8090:80 --address 0.0.0.0

10) Validation Commands (Quick Health Check)
kubectl -n lms get pods
kubectl -n lms get svc
kubectl -n lms logs deploy/web-server --tail=50
kubectl -n lms logs deploy/api-server --tail=50


Test internal API connectivity:

kubectl -n lms run curltest --rm -it --image=curlimages/curl -- \
  sh -lc 'curl -sS -i http://api-server:5000/api/courses | head -n 20'


  pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  parameters {
    booleanParam(name: 'FAIL_ON_TEST', defaultValue: false, description: 'Fail pipeline if tests fail')
    booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube scan + Quality Gate')
    booleanParam(name: 'FAIL_ON_SONAR', defaultValue: true, description: 'Fail pipeline if Sonar scan fails')
    string(name: 'K8S_NAMESPACE', defaultValue: 'lms', description: 'Kubernetes namespace')
    string(name: 'KIND_CLUSTER', defaultValue: 'kind', description: 'KIND cluster name')
    string(name: 'KUBE_CONTEXT', defaultValue: 'kind-kind', description: 'kubectl context name')
    string(name: 'WEB_NODEPORT', defaultValue: '30080', description: 'NodePort to access web-server')
  }

  environment {
    // Jenkins -> Global Tool Configuration -> NodeJS installation name
    NODEJS_TOOL = 'node20'

    // Jenkins -> Manage Jenkins -> System -> SonarQube servers -> Name
    SONARQUBE_SERVER = 'sonarqube'

    // OPTIONAL: Jenkins -> Global Tool Configuration -> SonarScanner installation name
    // If you don't have this tool, pipeline will fallback to npx sonar-scanner
    SONAR_SCANNER_TOOL = 'SonarScanner'

    API_IMAGE = 'lms-public-api'
    WEB_IMAGE = 'lms-web'
  }

  stages {

    stage('Checkout') {
      steps {
        cleanWs()
        checkout scm

        sh(label: 'versions', script: '''#!/usr/bin/env bash
          set -e
          echo "Shell: $SHELL"
          echo "Node: $(node -v 2>/dev/null || true)"
          echo "NPM : $(npm -v 2>/dev/null || true)"
          docker version >/dev/null 2>&1 && echo "Docker OK" || echo "Docker NOT available"
          kubectl version --client=true || true
          kind version || true
        ''')

        sh(label: 'list workspace', script: '''#!/usr/bin/env bash
          set -e
          ls -la
        ''')
      }
    }

    stage('Install Dependencies') {
      steps {
        script {
          def nodeHome = tool name: "${env.NODEJS_TOOL}", type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
          withEnv(["PATH+NODE=${nodeHome}/bin"]) {
            sh(label: 'npm ci (api + webapp)', script: '''#!/usr/bin/env bash
              set -euo pipefail
              node -v
              npm -v
              if [ -d "api" ]; then (cd api && npm ci); fi
              if [ -d "webapp" ]; then (cd webapp && npm ci); fi
            ''')
          }
        }
      }
    }

    stage('Build') {
      steps {
        script {
          def nodeHome = tool name: "${env.NODEJS_TOOL}", type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
          withEnv(["PATH+NODE=${nodeHome}/bin"]) {
            sh(label: 'npm run build', script: '''#!/usr/bin/env bash
              set -euo pipefail
              if [ -d "api" ]; then (cd api && npm run build --if-present); fi
              if [ -d "webapp" ]; then (cd webapp && npm run build --if-present); fi
            ''')
          }
        }
      }
    }

    stage('Test') {
      steps {
        script {
          def nodeHome = tool name: "${env.NODEJS_TOOL}", type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
          withEnv(["PATH+NODE=${nodeHome}/bin"]) {

            int status = sh(returnStatus: true, label: 'npm test (api + webapp)', script: '''#!/usr/bin/env bash
              set -euo pipefail
              status=0
              if [ -d "api" ]; then (cd api && npm test --if-present) || status=$?; fi
              if [ -d "webapp" ]; then (cd webapp && npm test --if-present) || status=$?; fi
              exit $status
            ''')

            if (status != 0) {
              echo "⚠️ Tests returned non-zero exit code: ${status}"
              if (params.FAIL_ON_TEST) {
                error("FAIL_ON_TEST=true → failing pipeline due to test failure")
              } else {
                echo "Continuing pipeline (FAIL_ON_TEST=false)"
              }
            }
          }
        }
      }
    }

    stage('SonarQube Scan') {
      when { expression { return params.RUN_SONAR } }
      steps {
        script {
          def nodeHome = tool name: "${env.NODEJS_TOOL}", type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
          withEnv(["PATH+NODE=${nodeHome}/bin"]) {
            withSonarQubeEnv("${env.SONARQUBE_SERVER}") {

              // Try Jenkins SonarScanner tool; if not found, fallback to npx sonar-scanner
              def scannerHome = null
              try {
                scannerHome = tool name: "${env.SONAR_SCANNER_TOOL}", type: 'hudson.plugins.sonar.SonarRunnerInstallation'
              } catch (ignored) {
                scannerHome = null
              }

              int sonarStatus = 0
              if (scannerHome) {
                withEnv(["PATH+SONAR=${scannerHome}/bin"]) {
                  sonarStatus = sh(returnStatus: true, label: 'sonar-scanner (tool)', script: '''#!/usr/bin/env bash
                    set -euo pipefail
                    sonar-scanner
                  ''')
                }
              } else {
                sonarStatus = sh(returnStatus: true, label: 'sonar-scanner (npx fallback)', script: '''#!/usr/bin/env bash
                  set -euo pipefail
                  npx --yes sonar-scanner
                ''')
              }

              if (sonarStatus != 0) {
                echo "❌ Sonar scan failed with exit code: ${sonarStatus}"
                if (params.FAIL_ON_SONAR) {
                  error("FAIL_ON_SONAR=true → failing pipeline due to sonar failure")
                } else {
                  echo "Continuing pipeline (FAIL_ON_SONAR=false)"
                }
              }
            }
          }
        }
      }
    }

    stage('Quality Gate') {
      when { expression { return params.RUN_SONAR && params.FAIL_ON_SONAR } }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build + Load into KIND') {
      steps {
        sh(label: 'docker build + kind load', script: '''#!/usr/bin/env bash
          set -euo pipefail
          TAG="${BUILD_NUMBER}"
          docker build -t ${API_IMAGE}:${TAG} api/
          docker build -t ${WEB_IMAGE}:${TAG} webapp/
          kind load docker-image ${API_IMAGE}:${TAG} --name "${KIND_CLUSTER}"
          kind load docker-image ${WEB_IMAGE}:${TAG} --name "${KIND_CLUSTER}"
        ''')
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh(label: 'kubectl apply + rollout', script: '''#!/usr/bin/env bash
          set -euo pipefail
          NS="${K8S_NAMESPACE}"
          TAG="${BUILD_NUMBER}"

          kubectl config use-context "${KUBE_CONTEXT}"
          kubectl get ns "${NS}" >/dev/null 2>&1 || kubectl create ns "${NS}"

          cp -f lms-api.yaml lms-api.deploy.yaml
          cp -f lms-web.yaml lms-web.deploy.yaml
          sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-api.deploy.yaml
          sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-web.deploy.yaml

          kubectl -n "${NS}" apply -f postgres.yaml
          kubectl -n "${NS}" apply -f lms-api.deploy.yaml
          kubectl -n "${NS}" apply -f lms-web.deploy.yaml

          kubectl -n "${NS}" rollout status deploy/api-server --timeout=180s
          kubectl -n "${NS}" rollout status deploy/web-server --timeout=180s

          kubectl -n "${NS}" get pods -o wide
          kubectl -n "${NS}" get svc
        ''')
      }
    }

    stage('App URL (KIND NodePort)') {
      steps {
        sh(label: 'print URL', script: '''#!/usr/bin/env bash
          set -euo pipefail
          NODE_IP="$(ip -4 a | awk '/inet / && $2 !~ /^127/ {print $2}' | head -n1 | cut -d/ -f1 || true)"
          echo "✅ App URL:"
          echo "   http://${NODE_IP}:${WEB_NODEPORT}"
          echo "Port-forward example:"
          echo "  kubectl -n ${K8S_NAMESPACE} port-forward svc/web-server 8090:80 --address 0.0.0.0"
        ''')
      }
    }
  }

  post {
    always {
      sh(label: 'k8s status (always)', script: '''#!/usr/bin/env bash
        set +e
        kubectl config use-context "${KUBE_CONTEXT}" >/dev/null 2>&1 || true
        kubectl -n "${K8S_NAMESPACE}" get pods -o wide || true
        kubectl -n "${K8S_NAMESPACE}" get svc || true
      ''')
      archiveArtifacts artifacts: '**/lms-*.deploy.yaml', allowEmptyArchive: true
    }
    success { echo "✅ Pipeline succeeded." }
    failure { echo "❌ Pipeline failed. Check stage logs above." }
  }
}



