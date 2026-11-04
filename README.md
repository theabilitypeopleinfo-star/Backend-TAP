# TAP Backend API

Express.js backend providing end-to-end encrypted endpoints (AES-256-GCM + RSA-OAEP) for contact enquiries and donation intents.

## Production Installation & Dependencies

Production environments must never install development dependencies or execute non-reproducible package resolutions. 

Always install dependencies using:
```bash
npm ci --omit=dev
```

This guarantees:
1. Exact, reproducible installation from `package-lock.json`.
2. Omission of devDependencies (test runners, linters, tooling), keeping the production image minimal and free of dev tooling vulnerabilities.
3. Verification that `package.json` and `package-lock.json` are in complete sync.

## Configuration & Deployment

### Reverse Proxy & Client IP Detection (`TRUST_PROXY_HOPS`)

The API enforces strict rate limiting using `express-rate-limit`. To prevent IP spoofing while correctly rate limiting client requests behind reverse proxies, Express `trust proxy` is configured via the `TRUST_PROXY_HOPS` environment variable.

- **Non-production environments (test/development)**: Defaults to `false` when unset (direct socket IP used).
- **Production (`NODE_ENV=production`)**: Server startup **fails fatally** if `TRUST_PROXY_HOPS` is unset. You must explicitly configure the number of trusted upstream proxy hops.

#### Recommended settings for common hosting platforms:

| Platform | Recommended `TRUST_PROXY_HOPS` | Explanation |
| :--- | :--- | :--- |
| **Render** | `1` | Render routes inbound traffic through a single reverse proxy load balancer before your container. |
| **Railway** | `1` | Railway uses an edge proxy layer in front of container deployments. |
| **Google Cloud Run** | `1` | Cloud Run services sit directly behind Google Front End (GFE) / Cloud Load Balancing. |
| **Cloudflare + Reverse Proxy** | `2` | Traffic flows through Cloudflare edge (hop 1) followed by an origin reverse proxy/ingress (hop 2). |
| **AWS ALB / ECS** | `1` | AWS Application Load Balancer terminates TLS and appends to `X-Forwarded-For`. |
| **Direct connection (No proxy)** | `false` | Disables proxy header trust; uses direct socket remote address. |

Example `.env` configuration:
```env
# Set to 1 for Render / Railway / Cloud Run
TRUST_PROXY_HOPS=1
```

### Firebase App Check Enforcement (`APP_CHECK_ENFORCE_*`)

Firebase App Check verifies that inbound traffic originates from authentic, verified client applications rather than unauthorized clients or automated scripts.

Enforcement is configured on a per-route basis via environment variables to allow staged rollout across web and mobile clients:

| Route | Default State | Environment Variable | Description & Rationale |
| :--- | :--- | :--- | :--- |
| `POST /api/contact` | **Enforced** (`true`) | `APP_CHECK_ENFORCE_CONTACT` | Web public contact and volunteer submissions. Protected by App Check reCAPTCHA tokens. Missing or invalid tokens return `401 Unauthorized`. |
| `POST /api/enquiry` | **Monitor Mode** (`false`) | `APP_CHECK_ENFORCE_ENQUIRY` | Mobile client enquiry submissions. Currently operates in monitor mode (token validated and logged if present, unblocked if missing) while `tap-mobile` App Check (DeviceCheck / Play Integrity) is prepared. Set to `true` when mobile App Check goes live. |
| `POST /api/donation-intent` | **Monitor Mode** (`false`) | `APP_CHECK_ENFORCE_DONATION_INTENT` | Mobile client donation intent creation. Operates in monitor mode until `tap-mobile` App Check is enabled. Set to `true` when mobile App Check goes live. (Also accepts `APP_CHECK_ENFORCE_DONATION`). |
| *Global Fallback* | `false` | `APP_CHECK_ENFORCE` | Global default fallback if a route-specific flag is not set. |

Example `.env` configuration:
```env
# Web route stays strictly enforced
APP_CHECK_ENFORCE_CONTACT=true

# Mobile routes stay in monitor mode until tap-mobile App Check ships
APP_CHECK_ENFORCE_ENQUIRY=false
APP_CHECK_ENFORCE_DONATION_INTENT=false
```

### Mobile API Attack Surface & Route Gating (`ENABLE_MOBILE_API`)

To minimize backend attack surface, mobile-specific routes are gated behind the `ENABLE_MOBILE_API` feature flag:

| Route | Status | Gating Flag | Description |
| :--- | :--- | :--- | :--- |
| `POST /api/enquiry` | **Gated** | `ENABLE_MOBILE_API=true` | Mobile enquiry submission. Defaults to disabled (`404 Not Found`) in production until the mobile application is deployed. |
| `GET /api/enquiry` | **Gated** | `ENABLE_MOBILE_API=true` | Admin retrieval of mobile enquiries. Gated behind `ENABLE_MOBILE_API` and requires admin claims. |
| `POST /api/donation-intent` | **Disabled** | `ENABLE_MOBILE_API=true` + Real Gateway | Entirely disabled (`503 Service Unavailable`) while online payments are not shipping. Does not create database records or return fabricated client secrets. |

## Production Environment Variables

The table below lists every environment variable utilized by the backend in production.

> [!IMPORTANT]
> Never commit real secrets or credentials to source control. In production, load sensitive variables (such as `RSA_PRIVATE_KEY` and `FIREBASE_PRIVATE_KEY`) directly from your platform's Secret Manager (e.g., Google Cloud Secret Manager, Firebase Secret Manager, AWS Secrets Manager).

| Name | Required / Optional | Placeholder Example | Where the Value Comes From |
| :--- | :--- | :--- | :--- |
| `NODE_ENV` | **Required** | `production` | Platform deployment setting. Must be set to `production` in live environments to enforce security guards. |
| `PORT` | **Optional** (defaults to `4000`) | `8080` | Assigned by container orchestrator (e.g. Cloud Run, Render, Railway, AWS ECS) or server operator. |
| `TRUST_PROXY_HOPS` | **Required** in production | `1` | Network infrastructure topology. Set to the number of upstream reverse proxy hops (`1` for Cloud Run/Render/Railway, `2` for Cloudflare + proxy). Server startup fails fatally if unset in production. |
| `FRONTEND_ORIGIN` | **Required** in production | `https://theabilitypeople.org,https://www.theabilitypeople.org` | Verified public domain(s) of the web frontend. Comma-separated list of exact HTTPS origins allowed for CORS. Rejects wildcards, HTTP, and localhost in production. |
| `ENABLE_MOBILE_API` | **Optional** (defaults to `false` in production) | `false` | Attack surface minimization flag. When `false` (default in production), mobile routes (`/api/enquiry`, `/api/donation-intent`) return 404. Set to `true` once mobile app client is launched. |
| `RSA_PRIVATE_KEY` | **Required** in production | `"-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQC...\n-----END PRIVATE KEY-----\n"` | Secret Manager (Google Secret Manager, AWS Secrets Manager, GitHub Secrets). 3072-bit PKCS#8 RSA private key generated outside the repo using `node scripts/gen-prod-rsa-key.js <path>`. The public key is derived automatically in-memory. |
| `FIREBASE_PROJECT_ID` | **Required** in production | `tap-organization-prod` | Firebase Console -> Project Settings -> General -> Project ID. |
| `FIREBASE_CLIENT_EMAIL` | **Required** in production | `firebase-adminsdk-prod@tap-organization-prod.iam.gserviceaccount.com` | Google Cloud Console -> IAM & Admin -> Service Accounts -> Firebase Admin SDK service account email. |
| `FIREBASE_PRIVATE_KEY` | **Required** in production | `"-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQD...\n-----END PRIVATE KEY-----\n"` | Secret Manager. Derived from the `private_key` field of your Firebase service account JSON key file with escaped newlines (`\n`). |
| `APP_CHECK_ENFORCE_CONTACT` | **Optional** (defaults to `true`) | `true` | Platform/app configuration. Controls App Check token verification for `POST /api/contact` (web contact and volunteer submissions). Set to `true` to reject unverified requests with 401. |
| `APP_CHECK_ENFORCE_ENQUIRY` | **Optional** (defaults to `false`) | `false` | Platform/app configuration. Controls App Check token verification for `POST /api/enquiry` (mobile enquiries). Stays in monitor mode (`false`) until native mobile App Check attestation is released. |
| `APP_CHECK_ENFORCE_DONATION_INTENT` | **Optional** (defaults to `false`) | `false` | Platform/app configuration. Controls App Check token verification for `POST /api/donation-intent` (mobile donation intents; also accepts alias `APP_CHECK_ENFORCE_DONATION`). Stays in monitor mode (`false`) until mobile App Check is released. |
| `APP_CHECK_ENFORCE` | **Optional** (defaults to `false`) | `false` | Platform/app configuration. Global fallback enforcement state for any endpoint without an explicit per-route flag. |


