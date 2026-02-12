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


