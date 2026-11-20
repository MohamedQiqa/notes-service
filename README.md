```html
<div align="center">
```

# 📝 Notes Service

`A secure, stateless REST API for creating and managing personal notes, built with Kotlin and Spring Boot.`

!\[Kotlin]\(https\://img.shields.io/badge/Kotlin-1.9.25-7F52FF?style=for-the-badge\&logo=kotlin\&logoColor=white)
!\[Spring Boot]\(https\://img.shields.io/badge/Spring\_Boot-3.4.3-6DB33F?style=for-the-badge\&logo=springboot\&logoColor=white)
!\[Spring Security]\(https\://img.shields.io/badge/Spring\_Security-6DB33F?style=for-the-badge\&logo=springsecurity\&logoColor=white)
!\[Java]\(https\://img.shields.io/badge/Java-17-ED8B00?style=for-the-badge\&logo=openjdk\&logoColor=white)
!\[MongoDB]\(https\://img.shields.io/badge/MongoDB-47A248?style=for-the-badge\&logo=mongodb\&logoColor=white)

```html
</div>
```

## `Overview`

`Notes API provides the backend foundation for a multi-user notes application. It manages account registration and authentication, protects note operations with JWT-based authorization, and persists user-specific data in MongoDB.`

`The service is designed for a clear separation of responsibilities:`

- `REST controllers expose the public HTTP contract.`
- `Spring Security and JWT components enforce stateless authorization.`
- `MongoDB repositories handle users, notes, and refresh sessions.`
- `BCrypt protects passwords.`
- `GitHub Actions and Linux service configuration support automated deployment.`

## `What the Service Provides`

- `User registration with email format and password strength validation`
- `Login using BCrypt password verification`
- `Short-lived JWT access tokens`
- `Rotating, database-backed refresh tokens`
- `User-scoped note creation and retrieval`
- `Note updates through an ID-aware save operation`
- `Ownership-checked note deletion`
- `Stateless Spring Security configuration`
- `Global HTTP validation error responses`
- `MongoDB TTL cleanup for expired refresh tokens`
- `Automated packaging and deployment to a Linux server`

## `Technology Stack`

| `Area`             | `Technology`                     |
| ------------------ | -------------------------------- |
| `Language`         | `Kotlin 1.9`                     |
| `Runtime`          | `Java 17`                        |
| `API framework`    | `Spring Boot 3.4`                |
| `Security`         | `Spring Security, JJWT`          |
| `Password hashing` | `BCrypt`                         |
| `Data store`       | `MongoDB`                        |
| `Build system`     | `Gradle 8 with Kotlin DSL`       |
| `Testing`          | `JUnit 5, Spring Boot Test`      |
| `Reverse proxy`    | `Nginx`                          |
| `Deployment`       | `GitHub Actions, systemd, rsync` |

## `Architecture`

```text
Client
  |
  | HTTP + Bearer access token
  v
Nginx (reverse proxy)
  |
  v
Spring Boot REST API
  |
  +-- AuthController
  +-- NoteController
  +-- JwtAuthFilter
  +-- AuthService / JwtService
  |
  v
MongoDB
  +-- users
  +-- refresh_tokens
  +-- notes
```

`Protected endpoints require an access token in the Authorization header:`

```javascript
Authorization: Bearer <access-token>
```

`The /auth/** routes and the / status route are public. All other routes require authentication.`

## `Authentication Flow`

1. `A user registers with an email and password.`
2. `The password is hashed with BCrypt before it is stored.`
3. `Login validates the credentials and issues:`
   - `an access token valid for 15 minutes`
   - `a refresh token valid for 30 days`
4. `Protected requests are authenticated by a JWT filter.`
5. `Refresh tokens are hashed with SHA-256 and stored in MongoDB.`
6. `Using a refresh token deletes the consumed token and replaces it with a new pair.`
7. `MongoDB TTL indexing removes expired refresh-token records.`

`This design keeps raw refresh tokens out of the database and prevents reused refresh tokens from producing another token pair.`

## `API Reference`

### `Status`

| `Method` | `Endpoint` | `Authentication` | `Description`            |
| -------- | ---------- | ---------------- | ------------------------ |
| `GET`    | `/`        | `No`             | `Health/status response` |

`Example:`

```bash
curl http://localhost:8085/
```

`Response:`

```text
Everything cool!
```

### `Authentication`

| `Method` | `Endpoint`       | `Authentication` | `Description`                         |
| -------- | ---------------- | ---------------- | ------------------------------------- |
| `POST`   | `/auth/register` | `No`             | `Create an account`                   |
| `POST`   | `/auth/login`    | `No`             | `Authenticate and receive token pair` |
| `POST`   | `/auth/refresh`  | `No`             | `Rotate a refresh token`              |

#### `Register`

```bash
curl -X POST http://localhost:8085/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "person@example.com",
    "password": "StrongPass123"
  }'
```

`Password requirements:`

- `Minimum 9 characters`
- `At least one lowercase character`
- `At least one uppercase character`
- `At least one digit`

#### `Login`

```bash
curl -X POST http://localhost:8085/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "person@example.com",
    "password": "StrongPass123"
  }'
```

`Successful response:`

```json
{
  "accessToken": "<access-token>",
  "refreshToken": "<refresh-token>"
}
```

#### `Refresh`

```bash
curl -X POST http://localhost:8085/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{ "refreshToken": "<refresh-token>" }'
```

`The refresh endpoint validates the token type and signature, verifies that the token belongs to an existing user, and confirms that the hashed token is stored in MongoDB.`

### `Notes`

| `Method` | `Endpoint`    | `Authentication` | `Description`                             |
| -------- | ------------- | ---------------- | ----------------------------------------- |
| `POST`   | `/notes`      | `Bearer token`   | `Create or update a note`                 |
| `GET`    | `/notes`      | `Bearer token`   | `Retrieve the authenticated user's notes` |
| `DELETE` | `/notes/{id}` | `Bearer token`   | `Delete an owned note`                    |

`Create a note:`

```bash
curl -X POST http://localhost:8085/notes \
  -H "Authorization: Bearer <access-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "id": null,
    "title": "Project architecture",
    "content": "Define the service boundaries and deployment flow.",
    "color": 0
  }'
```

`Retrieve notes:`

```bash
curl http://localhost:8085/notes \
  -H "Authorization: Bearer <access-token>"
```

`Delete a note:`

```bash
curl -X DELETE http://localhost:8085/notes/<note-id> \
  -H "Authorization: Bearer <access-token>"
```

`The note response includes:`

```json
{
  "id": "<mongo-object-id>",
  "title": "Project architecture",
  "content": "Define the service boundaries and deployment flow.",
  "color": 0,
  "createdAt": "2026-01-01T12:00:00Z"
}
```

## `Data Model`

| `Collection`     | `Purpose`                                                       |
| ---------------- | --------------------------------------------------------------- |
| `users`          | `Email and BCrypt password hash`                                |
| `refresh_tokens` | `User relationship, expiration, and hashed refresh-token value` |
| `notes`          | `Note content, color, creation time, and owning user`           |

## `Local Setup`

### `Requirements`

- `JDK 17`
- `MongoDB instance`
- `Gradle Wrapper (included)`

### `1. Configure MongoDB`

`Start MongoDB locally or provide a MongoDB connection URI.`

### `2. Create a JWT signing secret`

`Generate a Base64-encoded 256-bit secret:`

```bash
openssl rand -base64 32
```

### `3. Set environment variables`

```bash
export MONGODB_CONNECTION_STRING="mongodb://localhost:27017/notes"
export JWT_SECRET_BASE64="<base64-encoded-secret>"
```

`The application reads both variables in application.properties.`

| `Variable`                  | `Purpose`                         |
| --------------------------- | --------------------------------- |
| `MONGODB_CONNECTION_STRING` | `MongoDB connection URI`          |
| `JWT_SECRET_BASE64`         | `Base64-encoded HMAC signing key` |

### `4. Run the service`

```bash
./gradlew bootRun
```

`The API starts on port 8085 by default.`

```text
http://localhost:8085
```

## `Build and Test`

`Run the test suite:`

```bash
./gradlew test
```

`Build an executable JAR:`

```bash
./gradlew bootJar
```

`The generated artifact is placed in:`

```text
build/libs/spring_boot_crash_course-0.0.1-SNAPSHOT.jar
```

`Run the JAR directly:`

```bash
java -jar build/libs/spring_boot_crash_course-0.0.1-SNAPSHOT.jar
```

## `Deployment`

`The repository includes deployment assets for a Linux host:`

- `.github/workflows/deploy.yml — builds and deploys on pushes to master`
- `deployment/notes.service — systemd service definition`
- `deployment/nginx-notes-config — Nginx reverse-proxy configuration`

`The deployment workflow:`

1. `Checks out the repository.`
2. `Loads an SSH private key from the DEPLOY_KEY GitHub secret.`
3. `Builds the executable JAR.`
4. `Transfers it to /opt/notes.`
5. `Restarts the notes.service systemd unit.`

`The server-side service reads runtime values from:`

```text
/etc/default/notes-env
```

`That file should define MONGODB_CONNECTION_STRING and JWT_SECRET_BASE64.`

`Before production use, configure your server's domain and TLS certificate in Nginx and replace the example server settings in the deployment workflow and Nginx configuration.`

## `Project Structure`

```text
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
```

## `Validation and Error Handling`

`Bean validation produces a structured 400 response for invalid registration or note payloads:`

```json
{
  "errors": [
    "Password must be at least 9 characters long and contain at least one digit, uppercase and lowercase character."
  ]
}
```

`Unauthorized protected requests receive HTTP 401. Invalid refresh tokens receive HTTP 401 with an explanatory response.`
