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


