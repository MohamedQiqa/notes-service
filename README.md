```markdown
<div align="center">📝 Notes Service

A secure, stateless REST API for creating and managing personal notes — built with Kotlin & Spring Boot.

<br/>Kotlin Spring Boot Spring Security Java <br/> MongoDB JWT Gradle GitHub Actions Nginx

<br/>Overview • Features • Architecture • Auth Flow • API • Setup • Deployment

</div>🔎 Overview

Notes Service is the backend foundation for a multi-user notes application. It handles account registration and authentication, protects note operations with JWT-based authorization, and persists user-specific data in MongoDB.

It is designed as a standalone service that any client — web, mobile, or another backend — can plug into over HTTP.

Layer

Responsibility

🌐 

REST Controllers

Expose the public HTTP contract

🛡️ 

Spring Security + JWT

Enforce stateless authorization

🗄️ 

MongoDB Repositories

Persist users, notes, and refresh sessions

🔐 

BCrypt

Protect passwords at rest

🚀 

GitHub Actions + systemd

Automate build and deployment

✨ Features

✅ User registration with email format and password strength validation

🔑 Login with BCrypt password verification

⏱️ Short-lived JWT access tokens (15 minutes)

🔄 Rotating refresh tokens (30 days), hashed and stored in the database

👤 User-scoped notes — each user only sees their own data

✏️ Create & update notes through a single ID-aware save operation

🗑️ Ownership-checked deletion

🧹 MongoDB TTL cleanup for expired refresh tokens

⚠️ Structured validation errors for clean client handling

🚀 Automated deployment to a Linux server

🧰 Tech Stack

Area

Technology

Language

Kotlin 1.9

Runtime

Java 17

Framework

Spring Boot 3.4

Security

Spring Security, JJWT

Password hashing

BCrypt

Database

MongoDB

Build

Gradle 8 (Kotlin DSL)

Testing

JUnit 5, Spring Boot Test

Reverse proxy

Nginx

Deployment

GitHub Actions, systemd, rsync

🏗️ Architecture

flowchart TD
    C[📱 Client<br/>Web / Mobile / Service] -->|HTTP + Bearer token| N[Nginx<br/>Reverse Proxy]
    N --> F[JwtAuthFilter]
    F --> AC[AuthController]
    F --> NC[NoteController]
    AC --> AS[AuthService / JwtService]
    AS --> DB[(MongoDB)]
    NC --> DB
    DB --- U[users]
    DB --- R[refresh_tokens]
    DB --- NO[notes]


/auth/** and / are public. All other routes require authentication:

Authorization: Bearer <access-token>


🔐 Authentication Flow

sequenceDiagram
    participant C as Client
    participant API as Notes Service
    participant DB as MongoDB

    C->>API: POST /auth/register (email, password)
    API->>DB: Save user (BCrypt hash)

    C->>API: POST /auth/login
    API->>DB: Verify credentials
    API-->>C: accessToken (15 min) + refreshToken (30 days)
    API->>DB: Store SHA-256 hash of refresh token

    C->>API: GET /notes (Bearer accessToken)
    API-->>C: User's notes

    C->>API: POST /auth/refresh (refreshToken)
    API->>DB: Delete used token, store new hash
    API-->>C: New token pair


🛡️ Raw refresh tokens never touch the database, and a reused refresh token cannot produce another token pair.

📡 API Reference

Base URL: http://localhost:8085

Status

Method

Endpoint

Auth

Description

GET

/

❌

Health / status check

curl http://localhost:8085/
# Everything cool!


Authentication

Method

Endpoint

Auth

Description

POST

/auth/register

❌

Create an account

POST

/auth/login

❌

Authenticate and receive a token pair

POST

/auth/refresh

❌

Rotate a refresh token

<details> <summary><b>Register</b></summary>curl -X POST http://localhost:8085/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "person@example.com",
    "password": "StrongPass123"
  }'


Password rules: minimum 9 characters, with at least one lowercase letter, one uppercase letter, and one digit.

</details> <details> <summary><b>Login</b></summary>curl -X POST http://localhost:8085/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "person@example.com",
    "password": "StrongPass123"
  }'


{
  "accessToken": "<access-token>",
  "refreshToken": "<refresh-token>"
}


</details> <details> <summary><b>Refresh</b></summary>curl -X POST http://localhost:8085/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{ "refreshToken": "<refresh-token>" }'


Validates the token type and signature, confirms the user exists, and checks that the hashed token is stored in MongoDB.

</details>Notes

Method

Endpoint

Auth

Description

POST

/notes

🔒 Bearer

Create or update a note

GET

/notes

🔒 Bearer

Get the authenticated user's notes

DELETE

/notes/{id}

🔒 Bearer

Delete an owned note

<details> <summary><b>Create a note</b></summary>curl -X POST http://localhost:8085/notes \
  -H "Authorization: Bearer <access-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "id": null,
    "title": "Project architecture",
    "content": "Define the service boundaries and deployment flow.",
    "color": 0
  }'


</details> <details> <summary><b>Get notes</b></summary>curl http://localhost:8085/notes \
  -H "Authorization: Bearer <access-token>"


{
  "id": "<mongo-object-id>",
  "title": "Project architecture",
  "content": "Define the service boundaries and deployment flow.",
  "color": 0,
  "createdAt": "2026-01-01T12:00:00Z"
}


</details> <details> <summary><b>Delete a note</b></summary>curl -X DELETE http://localhost:8085/notes/<note-id> \
  -H "Authorization: Bearer <access-token>"


</details>🗄️ Data Model

Collection

Stores

users

Email and BCrypt password hash

refresh_tokens

Owning user, expiration, hashed token value

notes

Title, content, color, creation time, owning user

⚙️ Local Setup

Requirements

☕ JDK 17

🍃 MongoDB instance (local or Atlas)

🐘 Gradle Wrapper (included)

1. Start MongoDB

docker run -d --name mongo -p 27017:27017 mongo


Or use a MongoDB Atlas connection URI.

2. Generate a JWT secret

openssl rand -base64 32


3. Set environment variables

export MONGODB_CONNECTION_STRING="mongodb://localhost:27017/notes"
export JWT_SECRET_BASE64="<base64-encoded-secret>"


Variable

Purpose

MONGODB_CONNECTION_STRING

MongoDB connection URI

JWT_SECRET_BASE64

Base64-encoded HMAC signing key

4. Run

./gradlew bootRun


🟢 The API starts on http://localhost:8085

🧪 Build & Test

./gradlew test        # run tests
./gradlew bootJar     # build executable JAR


java -jar build/libs/spring_boot_crash_course-0.0.1-SNAPSHOT.jar


🚀 Deployment

flowchart LR
    P[git push to master] --> GA[GitHub Actions]
    GA -->|build JAR| B[bootJar]
    B -->|rsync over SSH| S[/opt/notes on server/]
    S --> SD[systemctl restart notes.service]
    SD --> NG[Nginx → live API]


File

Purpose

.github/workflows/deploy.yml

Builds and deploys on pushes to 

master

deployment/notes.service

systemd service definition

deployment/nginx-notes-config

Nginx reverse-proxy configuration

The server reads runtime values from /etc/default/notes-env, which must define MONGODB_CONNECTION_STRING and JWT_SECRET_BASE64. The SSH key is loaded from the DEPLOY_KEY GitHub secret.

⚠️ Before production, configure your domain and TLS certificate in Nginx and replace the example server settings.

📁 Project Structure

src/main/kotlin/com/plcoding/spring_boot_crash_course/
├── controllers/
│   ├── AuthController.kt
│   ├── NoteController.kt
│   └── StatusController.kt
├── database/
│   ├── model/
│   └── repository/
├── security/
│   ├── AuthService.kt
│   ├── HashEncoder.kt
│   ├── JwtAuthFilter.kt
│   ├── JwtService.kt
│   └── SecurityConfig.kt
├── GlobalValidationHandler.kt
└── SpringBootCrashCourseApplication.kt


⚠️ Error Handling

Invalid payloads return a structured 400 response:

{
  "errors": [
    "Password must be at least 9 characters long and contain at least one digit, uppercase and lowercase character."
  ]
}


Case

Status

Invalid request body

400 Bad Request

Missing / invalid access token

401 Unauthorized

Invalid refresh token

401 Unauthorized

<div align="center">Made with ❤️ using Kotlin & Spring Boot

</div>
```
