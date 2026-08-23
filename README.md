# 🛡️ Homelab Edge Gateway (Traefik v3 + Authelia SSO + LLDAP Directory + Fail2Ban)

Enterprise-grade, automated edge reverse proxy and centralized identity/user management gateway for the cluster.

---

## 🏛️ Ingress Architecture & Components

```
                                  [ Public Internet Ingress ]
                                               │
                              https://*.example.com (Ports 80/443)
                                               │
                                               ▼
                              ┌──────────────────────────────────┐
                              │       TRAEFIK V3.1 (zap-vps)     │
                              │  - Wildcard TLS (Cloudflare DNS) │
                              │  - Automated Port 80 -> 443      │
                              │  - Fail2Ban Brute-Force Defense  │
                              └────────────────┬─────────────────┘
                                               │
                              Is User Authenticated on *.example.com?
                                               │
                      ┌────────────────────────┴────────────────────────┐
                      │ (ForwardAuth)                                   │
                      ▼                                                 ▼
          ┌───────────────────────┐                         ┌───────────────────────┐
          │  AUTHELIA SSO (9091)  │                         │    BACKEND ROUTING    │
          │  - Visual Login Page  │                         │  - Homepage (3000)    │
          │  - 2FA TOTP / WebAuthn│                         │  - Dozzle Logs (8080) │
          │  - Group & Role ACLs  │                         │  - Beszel Hub (8090)  │
          │  - Session Cookies    │                         │  - Remote Swarm Nodes │
          └───────────┬───────────┘                         └───────────────────────┘
                      │ (LDAP Protocol)
                      ▼
          ┌───────────────────────┐
          │  LLDAP DIRECTORY (GUI)│
          │  - Port 17170 (Web UI)│
          │  - Port 3890 (LDAP)   │
          │  - SQLite Database    │
          │  - Users & Group Mgmt │
          └───────────────────────┘
```

---

## 🌐 Generic Dynamic Domain Interpolation (`ROOT_DOMAIN`)

The gateway is **100% generic and portable**. You do not need to hardcode domain names in configurations.

### ⚙️ How it Works:
When the stack boots, the **`storage-init`** container automatically generates active runtime configurations from the templates based on your `.env`:

1. **LDAP Base DN (`DOMAIN_DC`) Auto-Computation:**
   * It dynamically converts any domain to its LDAP DC hierarchy:
     * `bluewave.work` $\rightarrow$ `dc=bluewave,dc=work`
     * `myhomelab.io` $\rightarrow$ `dc=myhomelab,dc=io`
     * `example.com` $\rightarrow$ `dc=example,dc=com`
2. **Dynamic Configuration Generation:**
   * Injects the LDAP `base_dn` and session `cookies.domain` into `authelia/configuration.yml`.
   * Injects the ForwardAuth redirection URL into `dynamic/middleware.yaml`.
   * Generates Traefik router rules in `dynamic/routes.yaml` for TLS certificates.

---

## 💾 Standard Data & Storage Hierarchy

Persistent state is stored cleanly outside the Git repository in the standard homelab hierarchy:

```
/home/${SYSTEM_USER}/DATA/gateway/data/
├── traefik/
│   ├── acme.json              # Let's Encrypt Wildcard SSL certificate store (0600)
│   └── logs/                  # Traefik access & error logs
├── authelia/
│   ├── db.sqlite3             # Authelia 2FA keys, TOTP tokens and session database
│   └── notification.txt       # Local identity verification codes
└── lldap/
    └── users.db               # SQLite database containing all LDAP users, groups & credentials
```

---

## 🚀 Deployment via Arcane GitOps

1. Open **Arcane Cockpit** at `https://arcane.example.com`.
2. Click **Projects** $\rightarrow$ **New Project**.
3. Set:
   * **Name:** `gateway`
   * **Git Repository:** `https://github.com/medzarka/home-lab-gateway.git`
   * **Branch:** `main`
4. Add Environment Variables (from `.env.example`):
   ```env
   SYSTEM_USER=mgrsys
   DATA_DIR=/home/mgrsys/DATA
   CLOUDFLARE_API_TOKEN=your_cloudflare_dns_api_token
   ACME_EMAIL=admin@example.com
   ROOT_DOMAIN=example.com
   SHARED_NETWORK=shared_net
   LLDAP_JWT_SECRET=generate_with_openssl_rand_hex_32
   LLDAP_KEY_SEED=generate_with_openssl_rand_hex_32
   LLDAP_ADMIN_PASSWORD=your_secure_admin_password
   LLDAP_BASE_DN=dc=example,dc=com
   AUTHELIA_LDAP_USER=authelia
   AUTHELIA_LDAP_PASSWORD=generate_service_password_here
   ```
5. Click **Deploy**.

---

## 👤 User & Group Management with LLDAP GUI

User and group administration is managed through the built-in **LLDAP Web UI**:

* 🌐 **Web Interface:** `https://lldap.example.com`
* 🔑 **Initial Setup Credentials:** Username `authelia` with `AUTHELIA_LDAP_PASSWORD` (or your created admin account).
* 🛡️ **Permanent Service Isolation:** Authelia uses its dedicated `AUTHELIA_LDAP_USER` account in the background. You can change your personal passwords and create users anytime without breaking Authelia!

### 1. Creating Users & Groups in LLDAP:
1. Log in to `https://lldap.example.com` as `authelia` with your service password.
2. **Create Your Personal Account:** Under **Users**, create your account (e.g. `mgrsys`) and add it to group `lldap_admin` (and your custom groups).
3. **Create Groups:** Under **Groups**, create user categories:
   * `admins` — Full cluster and infrastructure access.
   * `family` — Media, photos, cloud storage, and dashboard.
   * `work` / `students` — AI development, Open-WebUI, and knowledge tools.
4. **Create Users:** Under **Users**, create family, students, or team accounts.

---

## 🔒 Granular Group-Based Access Control (ACLs)

You can restrict access to any subdomain based on LLDAP group membership in [`authelia/configuration.yml`](authelia/configuration.yml) under `access_control.rules`:

```yaml
access_control:
  default_policy: "deny"
  rules:
    # ----------------------------------------------------
    # 1. Public Endpoints & APIs (Bypass SSO)
    # ----------------------------------------------------
    - domain: "auth.example.com"
      policy: "bypass"

    - domain:
        - "photos.example.com"
        - "mycloud.example.com"
        - "seafile.example.com"
      resources:
        - "^/api/.*$"
        - "^/api2/.*$"
        - "^/seafhttp/.*$"
      policy: "bypass"

    # ----------------------------------------------------
    # 2. Admins Only (Requires Password + 2FA TOTP)
    # ----------------------------------------------------
    - domain:
        - "traefik.example.com"
        - "arcane.example.com"
        - "lldap.example.com"
        - "metrics.example.com"
      subject:
        - "group:admins"
      policy: "two_factor"

    # ----------------------------------------------------
    # 3. Family & Admins (Storage, Photos, Dashboard)
    # ----------------------------------------------------
    - domain:
        - "homelab.example.com"
        - "hub.example.com"
        - "photos.example.com"
        - "seafile.example.com"
      subject:
        - "group:family"
        - "group:admins"
      policy: "one_factor"

    # ----------------------------------------------------
    # 4. Students & Work (AI Tools & Research)
    # ----------------------------------------------------
    - domain:
        - "open-webui.example.com"
        - "litellm.example.com"
        - "qdrant.example.com"
      subject:
        - "group:students"
        - "group:work"
        - "group:admins"
      policy: "one_factor"
```

### 🛡️ What Happens During Access:
* **Allowed:** If the user is in the permitted group, Authelia allows the request and forwards identity headers (`Remote-User`, `Remote-Groups`, `Remote-Email`) to the upstream application.
* **Denied:** If the user does not belong to the permitted group, Authelia blocks the request and displays **`HTTP 403 Forbidden` (Access Denied)**.

---

## 🎨 Custom Branding & Logo

Custom login assets reside in `authelia/assets/`:
* `authelia/assets/logo.png` $\rightarrow$ Login card header logo.
* `authelia/assets/favicon.ico` $\rightarrow$ Browser tab icon.
