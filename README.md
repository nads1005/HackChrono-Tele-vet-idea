# Tele-Veterinary Support — MVP README

 Low-bandwidth, NGO-friendly tele-veterinary platform connecting rural farmers with veterinarians. Open-source prototype (MVP) for hackathon/demo.

---

## What this repo contains (concise)

* Farmer-facing PWA/mobile client (React or React Native) — `frontend/`
* Vet & NGO dashboards (web) — `frontend/` (shared)
* Backend API (Node.js + Express) — `backend/`
* Database schema & seed scripts — `backend/db/`
* Docker Compose for local dev (MongoDB + backend + frontend) — `docker-compose.yml`
* Demo scripts & sample data — `demo/`

---

## Design goals for the MVP

* **Offline-first / low-bandwidth:** SMS/USSD fallback, lightweight UI.
* **Simple workflows:** Register animal → submit case (photo + description) → vet responds → e-prescription & reminders.
* **NGO-ready:** Dashboard for triage and mobile team coordination.
* **Open-source & modular:** Easy to extend, integrates with external registries later.

---

## Folder structure

```
tele-vet-mvp/
├─ backend/
│  ├─ src/
│  │  ├─ controllers/
│  │  ├─ models/
│  │  ├─ routes/
│  │  ├─ services/   # SMS, email, storage helpers
│  │  ├─ utils/
│  │  └─ app.js
│  ├─ db/
│  │  ├─ seed/
│  │  └─ migrations/
│  ├─ Dockerfile
│  └─ package.json
├─ frontend/
│  ├─ mobile/        # React Native or PWA (Ionic/React)
│  ├─ web/           # Vet & NGO dashboards (React)
│  ├─ public/
│  └─ package.json
├─ demo/
│  ├─ demo_script.md
│  └─ sample_images/
├─ docker-compose.yml
├─ .env.example
└─ README.md
```

---

## Quick start (local, dev)

**Prereqs:** Node.js (>=16), Docker & docker-compose (recommended), yarn/npm

1. Clone the repo

```bash
git clone https://github.com/<org>/tele-vet-mvp.git
cd tele-vet-mvp
```

2. Copy env example and set values

```bash
cp .env.example .env
# edit .env to set keys (see below)
```

3. Start services via Docker Compose (recommended)

```bash
docker-compose up --build
```

4. Install & run frontend/backend locally (optional)

```bash
# backend
cd backend
npm install
npm run dev

# frontend
cd ../frontend/web
npm install
npm start
```

Open `http://localhost:3000` for web dashboard and the mobile app served via expo (if using React Native / PWA use the mobile folder instructions).

---

## Environment variables (.env.example)

```
# App
PORT=4000
NODE_ENV=development

# Database
MONGO_URI=mongodb://mongo:27017/televet

# Auth
JWT_SECRET=changeme
JWT_EXPIRES_IN=7d

# Storage (optional)
S3_BUCKET=
S3_REGION=
S3_ACCESS_KEY=
S3_SECRET_KEY=

# SMS (optional - for production)
SMS_PROVIDER=twilio
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# WebRTC / signaling (if used)
SIGNALING_SERVER_URL=

# Demo/test
SEED_ADMIN_EMAIL=admin@example.com
SEED_ADMIN_PASSWORD=admin123
```

---

## Minimal API spec (for frontend / integration)

> Base URL: `http://localhost:4000/api`

### Auth

* `POST /auth/register` — Register user (farmer/vet/ngo)

  * Body: `{ name, phone, role: 'farmer'|'vet'|'ngo', password }`
  * Response: `{ success, token, user }`

* `POST /auth/login`

  * Body: `{ phone, password }`
  * Response: `{ token, user }`

### Animals

* `POST /animals` — Register new animal

  * Body: `{ ownerId, species, name, ageMonths, tagId?, photos[] }`
  * Response: `{ animal }`

* `GET /animals/:ownerId` — List animals of owner

### Consultations

* `POST /consultations` — Farmer creates a consultation request

  * Body: `{ animalId, title, description, photos[] }`
  * Response: `{ consultationId, status: 'pending' }`

* `GET /consultations/:id` — Get consultation details (messages, status, vetId)

* `POST /consultations/:id/message` — Send message on a consultation (farmer/vet)

  * Body: `{ senderId, text?, mediaUrl?, type: 'text'|'photo'|'voice' }`

* `POST /consultations/:id/prescription` — Vet issues prescription

  * Body: `{ vetId, meds: [{name, dosage, days}], notes }`

### Vets

* `GET /vets/available?region=xyz` — List available vets for a region

### SMS webhook (if using SMS)

* `POST /sms/webhook` — Provider posts incoming SMS (map to farmer accounts)

---

## Data Models (simplified)

**User** (farmers/vets/ngos)

```json
{
  "_id": "",
  "name": "",
  "phone": "",
  "role": "farmer|vet|ngo",
  "credentials": { "licenseNumber": "" },
  "location": { "lat": 0, "lng": 0, "district": "" }
}
```

**Animal**

```json
{
  "_id": "",
  "ownerId": "userId",
  "species": "cow/goat/buffalo/etc",
  "name": "",
  "ageMonths": 24,
  "tagId": "optional",
  "photos": [],
  "records": [ { "date":"", "type":"vaccination/illness/treatment", "notes":"" } ]
}
```

**Consultation**

```json
{
  "_id": "",
  "animalId": "",
  "ownerId": "",
  "vetId": "optional",
  "status": "pending|open|closed|referred",
  "messages": [ { senderId, text?, mediaUrl?, createdAt } ],
  "prescription": { meds: [{name, dosage}], notes }
}
```

---

## SMS & USSD fallback

* Use a provider (Twilio / Exotel / Textlocal) in production. For the hackathon demo, you can simulate incoming SMS via a simple POST to `/api/sms/webhook` with `{ from, text }` and map `from` to farmer phone.
* USSD requires carrier integration; for MVP show a mock USSD flow in the demo (screenshots) or use a simple SMS keyword flow (e.g. `CARE <tag>`).

---

## Storage for photos & media

* For quick prototyping, store uploads locally (dev) or use a free S3-compatible service (e.g., DigitalOcean Spaces) in production.
* Keep file sizes small (compress images client-side) to save bandwidth.

---

## Demo script (3–5 minute live demo)

1. **Seed users**: Create one farmer (Ramu) and one vet (Dr. Sita) using seed script or register flows.
2. **Register an animal**: Ramu registers "Bholu" (cow) with a photo.
3. **Raise consultation**: Ramu uploads a photo + short voice note describing fever/lethargy and submits a consultation.
4. **Vet responds**: In the vet dashboard, Dr. Sita opens the request, types a diagnosis, issues a prescription and schedules a follow-up.
5. **Show SMS**: Demo SMS reminder for medicine dosage (simulated via webhook).
6. **Show record**: Open animal record to show consultation & prescription stored.

> Tip: Record a short screencapture beforehand and have the live demo steps ready if internet is flaky.

---

## Seed scripts & sample data

Place simple JSON files under `backend/db/seed/` and provide a script `npm run seed` to load example farmer/vet/animal/consultation data. Keep photos in `demo/sample_images/` and reference them in the seed.

---

## Docker Compose (example)

```yaml
version: '3.8'
services:
  mongo:
    image: mongo:5
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db
    ports:
      - '27017:27017'

  backend:
    build: ./backend
    restart: on-failure
    environment:
      - MONGO_URI=mongodb://mongo:27017/televet
    ports:
      - '4000:4000'
    depends_on:
      - mongo

  frontend:
    build: ./frontend/web
    ports:
      - '3000:3000'
    depends_on:
      - backend

volumes:
  mongo_data:
```

---



## Contribution & license

* Open-source MIT license (or choose permissive license preferred by NGO partners).
* Contribution guide: open issues, pick up `good-first-issue`, follow branching rules, include tests for critical routes.

---


