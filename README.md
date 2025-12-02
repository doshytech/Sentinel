## Secure Go Web App with CSRF protection

### 1. Introduction

Secure Go Web App with CSRF protection is a secure, scalable web application backend designed to demonstrate enterprise-grade authentication and session management. It addresses the critical security challenge of protecting stateless REST APIs against Cross-Site Request Forgery (CSRF) attacks while maintaining the scalability benefits of JSON Web Tokens (JWT).

Unlike standard JWT implementations that often overlook CSRF vulnerabilities, this project implements a hybrid security model. It binds a unique CSRF secret to the user's encrypted session token, ensuring that requests are only processed if they originate from trusted, authenticated contexts. This architecture is particularly suitable for high-risk environments, such as financial technology (FinTech) or personal data management systems, where data integrity and user identity verification are paramount.

---

### 2. Architecture Overview

High-Level Design (HLD)

The system follows a modular, layered architecture designed for separation of concerns. It utilizes a custom middleware chain to intercept, validate, and sanitize requests before they reach the core business logic.

**System Component Flow:**

```
Client (Browser)
   │
   ├── (1) HTTP Request (w/ Cookies & Headers)
   ▼
[ HTTP Server & Router ]
   │
   ├── (2) Recovery Middleware (Panic Handling)
   │
   ├── (3) Auth Middleware (Security Gateway)
   │      ├── Validate Access Token (JWT)
   │      ├── Verify CSRF Token (Header vs. Claims)
   │      └── Token Refresh Logic (if Expired)
   ▼
[ Request Handlers ]
   │
   ├── (4) Logic Handlers (Login, Register, Restricted)
   ▼
[ Service Layer ]
   ├── JWT Service (RSA Signing & Parsing)
   └── Data Persistence (User & Token Store)
```

---

### 3. Low-Level Design (LLD) & Core Modules

**Middleware Chain (middleware package):**

* **Recovery Handler:** Ensures system stability by catching run-time panics and preventing server crashes.
* **Auth Handler:** The security core. It inspects the AuthToken (JWT) and RefreshToken cookies. Crucially, it extracts the CSRF secret embedded within the JWT claims and compares it against the `X-CSRF-Token` provided in the request header. This "Double Submit" validation prevents unauthorized command execution.

**JWT Manager (myJwt package):**

* Manages the lifecycle of Access and Refresh tokens.
* Uses Asymmetric Cryptography (RSA-256) for signing tokens, ensuring that even if the verification key is leaked, new valid tokens cannot be forged without the private key.
* Handles silent token rotation: If an Access Token is expired but the Refresh Token is valid, a new Access Token is issued automatically without forcing a user logout.

**Data Persistence (db package):**

* Currently implemented as an in-memory store for high-performance demonstration.
* Includes logic for UUID generation, Bcrypt password hashing, and token revocation lists.

---

### 4. Use Cases & Real-World Applications

This architecture is specifically engineered for scenarios where security cannot be compromised for convenience.

* **FinTech & Banking Platforms:**

  * *Applications requiring strict adherence to security compliance (e.g., PCI-DSS, SOC2) can leverage the CSRF-bound JWT approach to prevent attackers from triggering unauthorized transactions on behalf of a logged-in user.*

* **Healthcare Portals (HIPAA Compliant):**

  * *Protecting sensitive patient data requires robust session management. The use of short-lived Access Tokens (15 minutes) minimizes the attack window if a session is hijacked.*

* **Enterprise SaaS Dashboards:**

  * *For B2B applications with multi-role access (Admin vs. User), the embedded Role claims in the JWT allow for efficient, stateless role-based access control (RBAC) without constant database lookups.*

---

### 5. Technical Details

**Tech Stack**

* Language: Go (Golang) 1.16+
* Authentication: JSON Web Tokens (JWT) using jwt-go
* Cryptography: * Signing: RSA-256 (Asymmetric Key Pair)
* Hashing: Bcrypt (Cost-factor adjustable)
* Secure Randoms: crypto/rand for high-entropy salt and token generation
* Frontend: HTML5 templates with jQuery/AJAX for asynchronous token handling.

**Key Architectural Choices**

* **Stateful vs. Stateless Hybrid:** While JWTs are stateless, we implement a Refresh Token Rotation strategy backed by a server-side store (the refreshTokens map). This allows us to revoke access immediately (e.g., "Log out all devices")—a feature often missing in pure JWT implementations.
* **CSRF Binding:** By embedding the CSRF secret inside the encrypted JWT TokenClaims, the server avoids storing CSRF tokens in a separate database table. The token validates itself, maintaining the scalability of the application.
* **Strict Cookie Policy:** Tokens are stored in HttpOnly cookies, making them inaccessible to client-side JavaScript (mitigating XSS attacks), while the CSRF token is sent via a custom header (mitigating CSRF attacks).

---

### 6. Setup & Running the Project

**Prerequisites**

* Go (version 1.16 or higher)
* OpenSSL (for generating RSA keys)

**Installation**

1. **Clone the repository:**

```bash
git clone https://github.com/yourusername/GoSecureAuth.git
cd GoSecureAuth
```

2. **Generate RSA Keys:**
   Critical Step: The application requires a private/public key pair to sign and verify JWTs. Create a `keys` directory in the root folder and generate the keys:

```bash
mkdir keys
openssl genrsa -out keys/app.rsa 4096
openssl rsa -in keys/app.rsa -pubout > keys/app.rsa.pub
```

3. **Install Dependencies:**

```bash
go mod download
```

4. **Run the Application:**

```bash
go run main.go
```

The server will start listening on `localhost:9000`.

**Verifying the Flow**

* Navigate to `http://localhost:9000/register` to create a user.
* Upon success, you will be redirected and tokens (Auth & Refresh) will be set in your cookies.
* Navigate to `http://localhost:9000/restricted` to verify access to protected routes.
* Check the server logs to see the middleware validating the CSRF token and refreshing the JWTs automatically.

---

### 7. Additional Enhancements

While this project is production-ready in its architectural logic, the following improvements would be recommended for a large-scale deployment:

* **Persistent Storage:** Migrate the `db` package from in-memory maps to a persistent database like PostgreSQL or MongoDB to ensure data survives server restarts.
* **Distributed Caching:** Use Redis to store Refresh Tokens. This allows for faster lookups and supports expiration events (TTL) natively.
* **Environment Configuration:** Move hardcoded configurations (ports, token expiration times) into a `.env` file or configuration struct for better DevOps practices.
* **Containerization:** Add a `Dockerfile` and `docker-compose.yml` to orchestrate the application and database containers.

---
