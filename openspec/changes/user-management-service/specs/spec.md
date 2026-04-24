# Specification: User Management Service

## Purpose
The User Management Service SHALL be responsible for all user identity, authentication, authorization, and profile management within the Medication Management System.

### Requirement 1: User Lifecycle Management
#### Scenario: Registration/Onboarding
- **Given** a new user provides mandatory registration details,
- **When** the registration API is called with valid data,
- **Then** the service SHALL create a new user record, issue a verification token, and store the profile in the PostgreSQL user database with encryption at rest.

#### Scenario: Login/Authentication
- **Given** a user presents correct credentials or approved SSO flow,
- **When** the authentication API is called,
- **Then** the service SHALL validate credentials (via OAuth 2.0), generate a session token (JWT), and issue it to the client, storing short-lived session data in Redis.

#### Scenario: Profile Management
- **Given** an authenticated request with a corresponding JWT,
- **When** profile update API is invoked,
- **Then** the service SHALL update the record, audit the change, and enforce data validation and authorization checks.

### Requirement 2: Authorization and Roles
#### Scenario: Role-Based Access Control
- **Given** a user with an assigned role (patient, provider, admin, emergency),
- **When** any protected API is accessed,
- **Then** the service SHALL enforce permissions and return error 403 if unauthorized.

### Requirement 3: Accessibility and Preferences
#### Scenario: Accessibility Preferences
- **Given** an authenticated user requests accessibility settings changes,
- **When** preferences API is invoked,
- **Then** the service SHALL update preferences and ensure accessibility options are persisted and applied per WCAG 2.1 AA.

### Requirement 4: Audit and Compliance
- **Given** any profile, authentication, or permission change,
- **When** the change occurs,
- **Then** a tamper-evident audit record SHALL be generated and stored in Elasticsearch (or as specified).

### Requirement 5: Integration and Security
- Authentication SHOULD integrate with third-party providers (OAuth2/OpenID/SAML).
- Only strong/passphrase credentials SHALL be permitted.
- Tokens MUST be short-lived, revocable, and signed.
- All APIs SHALL validate input and protect against OWASP Top 10 flaws.

## Technologies & Runtime Stack
- Backend: Spring Boot (Java)
- DB: PostgreSQL (users/credentials)
- Cache: Redis (sessions, tokens)
- Auth: OAuth 2.0, OpenID Connect
- Deployed as Kubernetes container (AWS EKS)
- Encrypted at rest (AES-256)
- Observability: Prometheus metrics, ELK-based logging, Jaeger tracing

## Key Components
- **Controller Layer**: Exposes REST APIs for user registration, login, logout, profile, roles, preferences
- **Service Layer**: Implements business logic for identity and access management
- **Repository Layer**: Reads/writes from PostgreSQL/Redis
- **External Clients**: OAuth provider adapters

## APIs (Endpoint List)
- `POST /users/register` (register user)
- `POST /users/login` (authenticate user)
- `POST /users/logout` (invalidate session)
- `GET /users/me` (fetch profile)
- `PUT /users/me` (update profile/preferences)
- `POST /users/reset-password` (initiate password reset)
- `GET /users/roles` (list roles)
- `POST /users/roles/assign` (assign role)

## Data Models
- **User**
  - id: UUID
  - email: string (unique)
  - password_hash: string
  - role: enum (patient, provider, admin, emergency)
  - accessibility_prefs: JSON
  - created_at: timestamp
  - updated_at: timestamp

- **Session**
  - session_id: UUID
  - user_id: UUID
  - jwt: string
  - expires_at: timestamp

## Interactions with Dependencies
- **OAuth Provider:** HTTP(S) (OAuth2/OpenID endpoints), performs redirect and token exchange, validates external identity tokens.
- **PostgreSQL:** JDBC connection, stores/encrypts user profiles/credentials.
- **Redis:** Caches session data, temporary tokens (auto-expiry).
- **Elasticsearch:** Receives audit logs.
- **API Gateway:** All client calls pass through, authenticates using session/JWT.

- **Error Handling:** Standardized error responses (400, 401, 403, 409, 500); retry logic for external auth calls.
- **Security:** Input validation, rate limiting via API Gateway, JWT revocation support.

## Key Flows
1. **Registration**
   1. User submits registration form.
   2. Controller validates and calls service.
   3. Service creates user in PostgreSQL (with password hashing/encryption).
   4. Verification token generated and sent (e.g., email via Notification Service).
   5. User’s role assigned (default: patient).
   6. Audit logged.

2. **Login**
   1. User submits credentials or SSO token.
   2. Controller/Service validates via OAuth/OpenID.
   3. JWT session token issued and stored in Redis.
   4. Client receives JWT for further requests.
   5. Audit logged.

3. **Profile Update**
   1. Authenticated user accesses `PUT /users/me`.
   2. Updates validated and persisted.
   3. Old/new values audit-logged.

4. **Role Management**
   1. Admin calls role assignment API.
   2. Service checks authorization.
   3. Role updated; change logged.

## TODO
- Explicit OpenAPI spec for endpoints
- Full data field list for accessibility_prefs
- Sequence diagrams for onboarding and OAuth flows

---