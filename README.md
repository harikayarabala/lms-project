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


