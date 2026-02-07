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


