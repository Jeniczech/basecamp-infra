# Basecamp 🏔️

Infrastructure configuration and recovery runbook for **Basecamp**, a small self-hosted server used to run personal web applications.

Basecamp currently provides shared:

- PostgreSQL
- Caddy reverse proxy
- HTTPS termination
- Docker networking

Individual applications are managed separately in their own repositories.

---

## Architecture

```text
Internet
   │
   │ :80 / :443
   ▼
┌──────────────────────────────────────────┐
│ Basecamp                                 │
│ Ubuntu 24.04                             │
│                                          │
│  Caddy                                   │
│    │                                     │
│    │ reverse proxy                       │
│    ▼                                     │
│  application containers                  │
│    │                                     │
│    ▼                                     │
│  PostgreSQL                              │
│                                          │
│  Docker network: basecamp-network        │
└──────────────────────────────────────────┘
```

Only Caddy is exposed to the internet on ports `80` and `443`.

Application ports and PostgreSQL are not published publicly.

---

## Repository contents

```text
.
├── compose.yml
├── Caddyfile
├── .env.example
├── .gitignore
└── README.md
```

### `compose.yml`

Defines shared Docker services:

- PostgreSQL
- Caddy

### `Caddyfile`

Defines public domains and reverse proxy routing to individual applications.

### `.env.example`

Documents environment variables required by the infrastructure.

Actual secrets must never be committed.

---

# Server layout

Shared infrastructure lives in:

```text
/opt/basecamp
```

Applications live separately:

```text
/opt/apps
```

Example:

```text
/opt/
├── basecamp/
│   ├── compose.yml
│   ├── Caddyfile
│   └── .env
│
└── apps/
    └── playground/
        ├── compose.yml
        └── .env
```

The files in this repository are the source of truth for the non-secret shared infrastructure configuration.

Production secrets and persistent application data are deliberately not stored in Git.

---

# Shared Docker network

All applications that need to communicate with Basecamp infrastructure join:

```text
basecamp-network
```

It is an external Docker network so it can be shared between independent Docker Compose projects.

For example:

```yaml
networks:
  basecamp-network:
    external: true
```

Applications can then reach PostgreSQL using the Docker hostname:

```text
postgres
```

Caddy can reach applications using their configured Docker network aliases.

---

# PostgreSQL

Basecamp runs one shared PostgreSQL server.

Each application should have:

- its own database
- its own PostgreSQL role
- its own password

Applications must not connect using the PostgreSQL superuser.

Example:

```text
PostgreSQL
├── playground database
│   └── playground role
│
├── app-a database
│   └── app-a role
│
└── app-b database
    └── app-b role
```

This design was chosen instead of running a PostgreSQL container for every application to reduce resource usage on the VPS.

PostgreSQL data is persisted in the Docker volume:

```text
postgres_data
```

The Docker volume protects the data when the PostgreSQL container itself is recreated.

It is **not a backup**.

---

# Caddy

Caddy acts as the public reverse proxy.

It:

- listens on ports `80` and `443`
- obtains and renews TLS certificates
- terminates HTTPS
- routes requests to application containers

For example:

```caddy
praguescavengerhunt.cz {
    reverse_proxy playground:3000
}

www.praguescavengerhunt.cz {
    redir https://praguescavengerhunt.cz{uri} permanent
}
```

The `playground` hostname is a Docker network alias, not a public DNS name.

---

# Secrets

The infrastructure requires:

```env
POSTGRES_PASSWORD=
```

Create the production file:

```text
/opt/basecamp/.env
```

from `.env.example` and supply the real value.

Restrict its permissions:

```bash
chmod 600 /opt/basecamp/.env
```

Never commit:

- `.env`
- database passwords
- SSH private keys
- GitHub tokens
- application secrets

---

# Disaster recovery / bootstrap

This section describes how to rebuild Basecamp starting from a clean Ubuntu 24.04 VPS.

It intentionally separates configuration stored in Git from secrets and persistent data stored elsewhere.

---

## 1. Provision the server

Start with a clean:

```text
Ubuntu 24.04
```

The current Basecamp size is:

```text
2 vCPU
4 GB RAM
40 GB SSD
```

The exact server size is not required by the configuration, but available resources should be considered when adding applications.

---

## 2. Create the administrator

Create the non-root administrator:

```bash
adduser jenda
usermod -aG sudo jenda
```

Configure the administrator's SSH public key in:

```text
/home/jenda/.ssh/authorized_keys
```

Verify that SSH key authentication works **before disabling password/root login**.

---

## 3. Harden SSH

Create:

```text
/etc/ssh/sshd_config.d/99-basecamp.conf
```

with:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Validate the SSH configuration before applying it.

Keep the existing SSH session open until a second connection using the `jenda` account and SSH key has been successfully tested.

---

## 4. Configure firewall

Enable UFW with incoming traffic denied by default.

Allow:

```text
22/tcp
80/tcp
443/tcp
```

The intended result is:

```text
SSH    → allowed
HTTP   → allowed
HTTPS  → allowed

everything else inbound → denied
```

Do not expose PostgreSQL port `5432`.

Application ports such as `3000` should not be exposed publicly either.

---

## 5. Configure fail2ban

Install and enable fail2ban.

At minimum, enable protection for:

```text
sshd
```

Verify that the service is running after configuration.

---

## 6. Install Docker

Install Docker Engine using Docker's official Ubuntu repository.

The installation should provide:

```text
Docker Engine
Docker Compose plugin
```

Add the administrator to the Docker group:

```bash
sudo usermod -aG docker jenda
```

Log out and back in for the group membership to take effect.

Verify:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

> Docker group membership effectively grants root-level control over the server. A dedicated restricted deployment user is a future hardening task.

---

## 7. Prepare directories

Create:

```bash
sudo mkdir -p /opt/basecamp
sudo mkdir -p /opt/apps
sudo chown jenda:jenda /opt/basecamp
sudo chown jenda:jenda /opt/apps
```

---

## 8. Restore Basecamp configuration

Clone this repository.

Copy or check out the infrastructure configuration into:

```text
/opt/basecamp
```

The directory should contain:

```text
/opt/basecamp/
├── compose.yml
├── Caddyfile
└── .env
```

The `.env` file is not stored in Git and must be restored separately.

---

## 9. Restore secrets

Create:

```text
/opt/basecamp/.env
```

containing:

```env
POSTGRES_PASSWORD=<production-secret>
```

Then:

```bash
chmod 600 /opt/basecamp/.env
```

Production secrets must come from the appropriate external secret/password storage and not from Git history.

---

## 10. Create the shared Docker network

The Compose configuration expects an external network:

```bash
docker network create basecamp-network
```

Verify:

```bash
docker network ls
```

This only needs to be created once.

---

## 11. Start shared infrastructure

From:

```bash
cd /opt/basecamp
```

start:

```bash
docker compose up -d
```

Verify:

```bash
docker compose ps
```

At this point PostgreSQL and Caddy should be running.

---

## 12. Restore PostgreSQL data

**TODO: backup and restore strategy has not yet been implemented.**

The current Docker volume provides persistence across container recreation, but it does not protect against:

- VPS loss
- disk failure
- accidental deletion
- corrupted data
- loss of the Docker volume

A proper recovery procedure will be documented here after automated PostgreSQL backups and tested restores are implemented.

For a completely lost server, this is currently the major missing piece of the disaster recovery process.

---

## 13. Restore applications

Each application is maintained in its own repository.

For every application:

1. create its `/opt/apps/<app>` directory
2. restore its production `.env`
3. create its PostgreSQL role/database if restoring into a fresh database
4. join `basecamp-network`
5. deploy its production Compose configuration
6. pull/deploy its application image
7. run required database migrations

Application-specific instructions belong in the application's own repository.

For example, the `playground` repository contains its production Compose configuration under:

```text
deploy/compose.yml
```

---

## 14. Configure DNS

Public application domains must point to the Basecamp public IPv4 address.

For an application domain, configure the appropriate DNS `A` record.

Do not modify unrelated:

- MX
- SPF
- DKIM
- DMARC
- mail/autodiscovery records

unless the mail configuration itself is intentionally being changed.

---

## 15. Verify Caddy and HTTPS

Once DNS points to Basecamp and the target application is reachable on `basecamp-network`, Caddy should obtain the required TLS certificate automatically.

Verify that:

```text
https://<domain>
```

is reachable.

Also verify any configured redirects such as:

```text
www → apex domain
```

---

# Adding a new application

A new application does not normally require another PostgreSQL or Caddy container.

## Database

Create a dedicated PostgreSQL role and database:

```sql
CREATE USER myapp WITH PASSWORD '<strong-password>';
CREATE DATABASE myapp OWNER myapp;
```

The application connection string will conceptually be:

```text
postgres://myapp:<password>@postgres:5432/myapp
```

---

## Application directory

Create:

```text
/opt/apps/myapp
```

Application-specific secrets should live in:

```text
/opt/apps/myapp/.env
```

and should not be committed.

---

## Docker network

Connect the application to:

```text
basecamp-network
```

Give it a stable network alias, for example:

```text
myapp
```

---

## Caddy

Add the domain to the version-controlled `Caddyfile`:

```caddy
example.com {
    reverse_proxy myapp:3000
}
```

Infrastructure configuration changes should be committed to this repository.

---

## DNS

Point the domain to the Basecamp public IP address.

Once DNS resolves correctly and the application is reachable, Caddy handles HTTPS certificate issuance.

---

# Source of truth

The intended ownership model is:

| Resource | Source of truth |
|---|---|
| Shared Docker Compose configuration | this repository |
| Caddy configuration | this repository |
| Required environment variable names | `.env.example` |
| Shared infrastructure secrets | external / server |
| Application code | application repository |
| Application production Compose | application repository |
| Application secrets | external / server |
| PostgreSQL data | database + backups |
| Server bootstrap procedure | this README |

Production configuration should not be changed manually on the server without reflecting the change in Git.

---

# Known limitations / future work

The current infrastructure works, but several production-hardening tasks remain.

High-priority items include:

- automated PostgreSQL backups
- tested PostgreSQL restore procedure
- CI test/lint quality gates before application deployment
- persistent/object storage for Payload Media
- post-deployment application health checks
- safe database migration/rollback strategy

Additional hardening:

- dedicated restricted deployment user
- dedicated read-only GHCR credentials on Basecamp
- monitoring and alerting
- production email configuration where required
- GitHub Actions deployment concurrency
- decide how shared infrastructure changes should be deployed from this repository

Full server provisioning is currently documented rather than automated. Ansible or similar tooling may be introduced later if maintaining the manual bootstrap becomes burdensome.

---

# Recovery status

If Basecamp is lost completely, the current recovery situation is:

```text
Server configuration        🟡 documented, mostly manual
Shared infrastructure       🟢 stored in Git
Application configuration   🟢 stored in Git
Application images          🟢 stored in GHCR
Application deployment      🟢 automated
Production secrets          🟡 external/manual recovery required
PostgreSQL data             🔴 backup strategy not implemented yet
Uploaded media              🔴 persistent storage strategy not implemented yet
```

The most important remaining disaster-recovery task is therefore PostgreSQL backup and restore.
