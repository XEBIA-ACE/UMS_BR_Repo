# Design: User Management Service

## Technical Approach
- **Microservice deployed on AWS EKS as a Spring Boot (Java) application.**
  - Follows hexagonal architecture: controller/service/repository layers.
- **REST endpoints** expose all user lifecycle, profile, and security APIs.
- **Separation of responsibilities**: Authentication logic, profile management, and access controls are isolated in dedicated service modules.

## Data Flow
- **Registration/Login:** Requests from Web/Mobile apps → API Gateway (Kong) → User Management Service → PostgreSQL/Redis/OAuth → Response.
- **Session management** handled via JWT tokens and Redis (short-lived; stateless scaling).
- **Profile updates or password resets**: Write directly to PostgreSQL (secured, encrypted).
- **Role changes:** Restricted via privileged API (enforced via claim checks).
- **Audit and compliance data** pushed asynchronously to Elasticsearch for search and reporting.

## Security
- **OAuth 2.0 (with PKCE for mobile), JWT for sessions.**
- **Strong password hashing**, MFA/biometric support (optional).
- **All APIs validate input; all sensitive events/updates are audit-logged.**

## Integration/Interactions
- **Direct dependencies:**
  - **PostgreSQL** (user/credential store, encrypted)
  - **Redis** (caching short-lived session data)
  - **OAuth/OpenID Providers** via REST APIs
- **Upstream/Downstream:**
  - **API Gateway**: All client traffic routes here; gateway enforces global security and rate-limit policy.
  - **Other services**: Rely on user claims via JWT or privileged RPC call.

## File/Component Changes
- `UserController` (REST API interface)
- `UserService` (business logic, orchestration)
- `UserRepository` (PostgreSQL CRUD)
- `SessionRepository` (Redis session store)
- `OAuthClient` (integrates with external providers)
- `AuditClient` (publishes to Elasticsearch/ELK)
- `API request/response DTOs` for all endpoints

---