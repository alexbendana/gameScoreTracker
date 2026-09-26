# Game Score Tracker

A web app for friends and colleagues to record game results against each other and track standings through a ranking system. Users can create or join groups, log matches with scores, view their stats and match history, and sort rankings by different criteria.


> ⏳ The backend runs on Render's free tier and sleeps when idle. The **first request can take up to ~50 seconds** while it wakes up. After that, it responds normally.

---

## Features

- **User authentication:** register, log in, and log out (JWT-based). Protected pages require login.
- **Groups:** create or join a group, rename it, toggle public/private visibility, and manage members.
- **Matches (CRUD):** add matches with scores, view match history per user, and delete matches.
- **Profiles & stats:** view your own profile and stats, view other players' profiles, update or delete your account.
- **Persistent storage:** all data is stored in a Supabase (PostgreSQL) database.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Angular 20, Angular Material, TypeScript, SCSS |
| Backend | Java 21, Spring Boot 3.5, Spring Security, Spring Data JPA / Hibernate |
| Auth | JSON Web Tokens (JWT) |
| Database | Supabase (PostgreSQL) in production, H2 in-memory database for local development |
| Hosting | Netlify (frontend), Render via Docker (backend) |
| Tooling | Git & GitHub, Maven, npm, AI coding assistants (used for deployment setup and configuration) |

---

## Architecture

```
┌──────────────────────┐   HTTPS + JWT   ┌──────────────────────┐   JDBC   ┌──────────────────────┐
│  Angular frontend    │ ──────────────▶ │  Spring Boot API     │ ───────▶ │  Supabase PostgreSQL │
│  (Netlify)           │ ◀────────────── │  (Render, Docker)    │ ◀─────── │                      │
└──────────────────────┘      JSON       └──────────────────────┘          └──────────────────────┘
```

- The frontend calls the REST API through services in `frontend-angular/src/app/services/`. `auth.interceptor.ts` attaches the JWT to each request, and `guards/auth.guard.ts` protects logged-in routes.
- The backend follows a **Controller → Service → Repository** structure for each feature (`auth`, `user`, `group`, `match`). `JwtAuthenticationFilter` and `SecurityConfig` handle authentication and CORS.
- Hibernate creates and updates the database tables automatically (`ddl-auto=update`).

### Project structure

```
gameScoreTracker/
├── backend-java/                  # Spring Boot REST API
│   ├── Dockerfile                 # Container build used by Render
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/cen4010/gamescoretracker/
│       │   ├── api/auth/          # Register & login
│       │   ├── api/user/          # User profile, stats, update, delete
│       │   ├── api/group/         # Group create/join/manage
│       │   ├── api/match/         # Match create/list/delete
│       │   ├── security/          # JWT filter, JWT utils, security + CORS config
│       │   └── utils/             # Exception handling, admin checks
│       └── resources/
│           ├── application.properties       # Local dev (H2 in-memory DB)
│           └── application-prod.properties  # Production (PostgreSQL via env vars)
├── frontend-angular/              # Angular single-page app
│   └── src/
│       ├── app/                   # Components, services, guards, routes
│       └── environments/          # API base URLs
└── netlify.toml                   # Netlify build + SPA redirect config
```

### API endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Create an account |
| POST | `/api/auth/login` | Log in and receive a JWT |
| GET | `/api/users/me` | Current user's profile |
| GET | `/api/users/{id}` | Another user's profile |
| PUT | `/api/users/me/update` | Update current user |
| DELETE | `/api/users/{id}` | Delete a user |
| GET | `/api/groups/all` | List groups |
| GET | `/api/groups/groupdetails` | Group details & rankings |
| POST | `/api/groups/join` | Join a group |
| PUT | `/api/groups/editname` | Rename a group |
| PUT | `/api/groups/togglevisibility` | Toggle group visibility |
| PUT | `/api/groups/manage` | Manage group members |
| POST | `/api/matches/add` | Record a match |
| GET | `/api/matches/user/{userId}` | A user's match history |
| DELETE | `/api/matches/{id}` | Delete a match |

---

## Running the Project Locally

### Prerequisites

- **Java 21 (JDK):** https://www.oracle.com/java/technologies/downloads/#jdk21-windows (on Windows, choose the x64 Installer)
- **Node.js 20.19+ or 22.12+** (includes npm): https://nodejs.org
- **Git:** https://git-scm.com
- *(Optional)* **IntelliJ IDEA Community** via [JetBrains Toolbox](https://www.jetbrains.com/toolbox-app/) and **VS Code** for the frontend
- Maven is **not** required. The project includes the Maven Wrapper (`mvnw`).

### 1. Clone the repository

```bash
git clone https://github.com/alexbendana/gameScoreTracker.git
cd gameScoreTracker
```

### 2. Start the backend

Locally, the backend uses an **H2 in-memory database**, so no database setup is needed.

```bash
cd backend-java

# macOS / Linux
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

The API starts at **http://localhost:8080**. You can inspect the local database at http://localhost:8080/h2-console (JDBC URL: `jdbc:h2:mem:testdb`, user: `sa`).

> Note: H2 is in-memory, so local data resets every time the backend restarts.

### 3. Start the frontend

In a **second terminal**:

```bash
cd frontend-angular
npm install
```

The frontend services import their API URL from `src/environments/environment.prod.ts`, which points to the deployed Render backend. **To use your local backend**, temporarily change `apiBaseUrl` in that file to:

```ts
apiBaseUrl: 'http://localhost:8080/api'
```

Then run:

```bash
npm start
```

Open **http://localhost:4200**. Register a new account, log in, and start creating groups and matches.

---

## Deployment

The app is deployed with three free-tier services.

### Database: Supabase

1. Create a project at https://supabase.com.
2. Click **Connect → Direct**, set **Connection Method** to **Session pooler** and **Type** to **JDBC**.
3. From the connection string, note the **URL** (everything before `?`), the **username** (`postgres.<project-id>`), and your database password.

Tables are created automatically by Hibernate on first startup.

### Backend: Render (Docker)

1. On https://render.com, create a **Web Service** from this repo.
2. Settings: **Root Directory** `backend-java`, **Language** `Docker`, **Instance Type** `Free`.
3. Add these environment variables:

| Variable | Value |
|---|---|
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `SPRING_DATASOURCE_URL` | Supabase JDBC URL (`jdbc:postgresql://...pooler.supabase.com:5432/postgres`) |
| `SPRING_DATASOURCE_USERNAME` | `postgres.<project-id>` |
| `SPRING_DATASOURCE_PASSWORD` | Supabase database password |
| `JWT_SECRET` | A long random string (32+ characters) |
| `CORS_ALLOWED_ORIGINS` | Your Netlify URL, e.g. `https://thegamescoretracker.netlify.app` |
| `JAVA_TOOL_OPTIONS` | `-XX:MaxRAMPercentage=75 -XX:+UseSerialGC -XX:TieredStopAtLevel=1 -Xss512k` (reduces memory use on the free tier) |

### Frontend: Netlify

1. Set `apiBaseUrl` in `frontend-angular/src/environments/environment.prod.ts` to your Render URL plus `/api`.
2. On https://netlify.com, import this repo. The build settings are read from `netlify.toml`:
   - **Base directory:** `frontend-angular`
   - **Build command:** `npm run build`
   - **Publish directory:** `dist/frontend-angular/browser`
3. `netlify.toml` also includes a redirect so Angular routes (e.g. `/profile`) work on page refresh.
4. To stay within Netlify's free limits, builds are stopped after the final deploy (**Project configuration → Build & deploy → Build status → Stopped builds**).

---
