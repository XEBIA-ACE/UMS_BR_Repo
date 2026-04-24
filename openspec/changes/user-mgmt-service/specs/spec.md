# User Management Service – Technical Specification

## Purpose

This service provides secure, reliable user authentication and management, with advanced features supporting accessibility and emergency scenarios in the medication management system.

---

### Requirement: Registration and Authentication

#### Scenario: Registering a New User

- **Given** a RegistrationRequest with all required fields (email, password, firstName, lastName)
- **When** the POST /api/v1/auth/register endpoint is called
- **Then** the service SHALL create a new user, return a valid AuthResponse, and publish a UserCreatedEvent.

#### Scenario: Logging In

- **Given** a valid email and password (LoginRequest)
- **When** POST /api/v1/auth/login is called
- **Then** the service SHALL authenticate the user, issue a JWT access token, and return user info in AuthResponse.

#### Scenario: Refreshing an Access Token

- **Given** a valid refresh token
- **When** POST /api/v1/auth/refresh is called
- **Then** the service SHALL issue a new access token with identical or reduced scope.

### Requirement: Profile Management

#### Scenario: Updating User Profile

- **Given** an authenticated, authorized user
- **When** PUT /api/v1/users/{userId} is called with valid payload
- **Then** the service SHALL update the profile, persist changes, and publish a UserUpdatedEvent.

### Requirement: Preferences Management

#### Scenario: Accessibility and Language Preferences

- **Given** a valid UserPreferencesRequest
- **When** PUT /api/v1/users/{userId}/preferences is invoked
- **Then** preferences (language, accessibility, notifications) SHALL be saved, available on retrieval, and used in personalization flows.

### Requirement: Emergency Access

#### Scenario: Generate Emergency Code

- **Given** valid user and access enabled
- **When** POST /api/v1/users/{userId}/emergency-access/code is called
- **Then** the service SHALL generate a time-limited code, store audit log, and return EmergencyAccessCodeResponse.

#### Scenario: Emergency Info Retrieval

- **Given** a valid or expired/invalid code
- **When** GET /api/v1/emergency-access/{code} is called
- **Then** service SHALL return user emergency info if code valid, else error (400/404/410).

### Requirement: Security and Compliance

- All sensitive data SHALL be encrypted at rest.
- OAuth 2.0 with JWT SHALL be enforced for all endpoints except public/health/open registration/login.
- Audit logging (security events) SHALL occur on profile, settings, and emergency access changes.
- Rate limiting and brute force protection SHALL be active per authentication events.

### Requirement: Observability

- Liveness/readiness at /api/v1/health, metrics at /api/v1/metrics, and structured logging MUST be present.

---

### Technology and Runtime Stack

- Spring Boot
- OAuth 2.0, JWT
- PostgreSQL (user, profile, settings)
- Redis (sessions, cache)
- Kafka (event publishing)

---

### Components

- Controllers: Auth, User, Preferences, EmergencyAccess, Accessibility
- Services: AuthenticationService, UserApplicationService, UserPreferencesService, SecurityService, LocalizationService

---

### Endpoints Summary

- POST /api/v1/auth/register, /login, /refresh, /logout, /password/reset-request, /password/reset
- GET/PUT/DELETE /api/v1/users, /users/{userId}, /users/{userId}/preferences, /users/{userId}/emergency-access
- GET /api/v1/emergency-access/{code}
- GET /api/v1/health, /metrics

---

### Data Models

- User (id, email, firstName, lastName, dateOfBirth, phoneNumber, status, etc.)
- UserPreferences (language, accessibility: highContrast, largeText, screenReader, notifications)
- EmergencyAccess (enabled, trustedContacts, emergencyInformation, accessCodeDuration)

---

### Interactions & Error Handling

- On user creation/update, relevant events MUST be published to Kafka.
- Redis cache/Miss fallback to DB for session/data as needed.
- Notifications sent via Notification Client REST calls.
- Error responses: Structured ErrorResponse (status, error, message, timestamp, path, traceId).
- Retry/backoff on event/notification failures.
- Security exceptions for forbidden/unauthorized, validation errors for invalid input with violations.

---

### Key Flows

1. **User Registration Flow**
   1. Validate RegistrationRequest.
   2. Hash password, store user in PostgreSQL.
   3. Assign default roles.
   4. Generate JWT, return AuthResponse.
   5. Publish UserCreatedEvent to Kafka.

2. **Authentication Flow**
   1. Validate credentials, check account status.
   2. Issue JWT, store session in Redis.
   3. On failure, increase login attempt count and return error.

3. **Preferences Management**
   1. Authenticated user submits preferences update.
   2. Update DB, cache, and publish PreferencesChanged event.

4. **Emergency Access Flow**
   1. User enables emergency access/adds contacts.
   2. User or system issues code (with expiry).
   3. Emergency responder presents code, system validates, logs event, returns medical info.

5. **Error Handling**
   1. Bad input: Return ValidationErrorResponse (HTTP 400).
   2. Unauthorized/forbidden: HTTP 401/403.
   3. Not found: HTTP 404.
   4. Rate limit: HTTP 429.

---

### TODO

- List exact permission granularity and roles.
- Confirm notification integration payloads with Notification Service.
- Provide sequence diagram for account recovery flow.

---