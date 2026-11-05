# Self-Hosted Nextcloud on Oracle Cloud Free Tier

A personal cloud storage server built with **Nextcloud, Docker Compose, Caddy, and Oracle Cloud Infrastructure (OCI)**.

The idea is simple: host my own private cloud storage instead of depending entirely on third-party cloud storage providers.

This repository documents two deployment approaches:

- **Standard deployment:** PostgreSQL + Redis + Nextcloud + Cron + Caddy.
- **Low-resource deployment:** SQLite + Nextcloud + Caddy, with reduced resource consumption.

I'm currently running the low-resource configuration on an Oracle Cloud Free Tier VM.

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Cloud Provider | Oracle Cloud Infrastructure – Free Tier |
| Operating System | Ubuntu 22.04 LTS |
| Cloud Storage | Nextcloud |
| Containerization | Docker & Docker Compose |
| Web Server | Apache (Nextcloud Apache image) |
| Reverse Proxy | Caddy |
| HTTPS | Let's Encrypt |
| Dynamic DNS | DuckDNS |
| Standard Database | PostgreSQL or MariaDB |
| Low-resource Database | SQLite |
| Cache | Redis (standard deployment) |
| Firewall | UFW / iptables / nftables |

## 🏗️ Architecture

### Standard deployment

```text
                    Internet
                       |
                       v
                 DuckDNS Domain
                       |
                       v
                 Oracle Cloud VM
                       |
                 Firewall Rules
                       |
                   Ports 80/443
                       |
                       v
                      Caddy
                 (Reverse Proxy)
                       |
                 localhost:8080
                       |
                       v
                  Nextcloud
                       |
              +--------+--------+
              |        |        |
              v        v        v
          PostgreSQL  Redis    Cron
              |
              v
        Persistent Storage
```

### Low-resource deployment

```text
                    Internet
                       |
                       v
                 DuckDNS Domain
                       |
                       v
                 Oracle Cloud VM
                       |
                 Firewall Rules
                       |
                   Ports 80/443
                       |
                       v
                      Caddy
                       |
                 localhost:8080
                       |
                       v
                  Nextcloud
                       |
                       v
                    SQLite
                       |
                       v
              Persistent Storage
```

Caddy handles incoming HTTP/HTTPS traffic and forwards requests to the Nextcloud container.

Nextcloud is not directly exposed to the internet. Its HTTP port is bound to localhost on the host machine.

---

# ☁️ 1. Oracle Cloud Setup

I created an Oracle Cloud Infrastructure account and used its Always Free eligible resources.

### My VM configuration

| Configuration | Value |
|---|---|
| Cloud Provider | Oracle Cloud |
| Instance Type | VM.Standard.E2.1.Micro |
| Architecture | AMD x86_64 |
| CPU | 1 OCPU |
| RAM | Approximately 1 GB |
| Operating System | Ubuntu 22.04.5 LTS |
| Boot Volume | 49 GB |
| Swap | 2 GB |

I initially attempted to provision an Ampere A1 instance, which provides ARM-based compute resources, but encountered an out-of-capacity error in my availability domain.

I eventually went with the AMD E2.1.Micro instance.

Since this VM has limited RAM, I deliberately chose a lightweight Nextcloud configuration.

**Note:** Oracle Free Tier resource eligibility, quotas and availability can change. Check the current terms before creating your VM.

Useful resource: [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)

---

# 🌐 2. Domain Setup with DuckDNS

I didn't purchase a domain for this project.

Instead, I registered a free subdomain using [DuckDNS](https://www.duckdns.org/).

Example:

```text
your-domain.duckdns.org
```

I configured the DNS record to point to my Oracle VM's public IP address.

Verify DNS resolution:

```bash
nslookup your-domain.duckdns.org
```

The returned IP address should match your VM's public IP.

If your public IP changes, update the DuckDNS record accordingly. You can also configure a DuckDNS update client to keep it synchronized automatically.

---

# 🐳 3. Install Docker and Docker Compose

Update the system:

```bash
sudo apt update
sudo apt upgrade -y
```

Install Docker using the official installation instructions:

[Docker Engine Installation – Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

Verify the installation:

```bash
docker --version
docker compose version
```

Docker Compose will be used to manage the Nextcloud application and its supporting services.

---

# 📁 4. Create the Project Directories

Create a directory for the deployment configuration:

```bash
sudo mkdir -p /opt/nextcloud
sudo mkdir -p /srv/nextcloud/data
```

The directory structure will look like this:

```text
/opt/nextcloud/
├── compose.yaml
├── .env
└── apache-memory.conf

/srv/nextcloud/
└── data/
```

- `/opt/nextcloud`: Docker Compose configuration and environment variables.
- `/srv/nextcloud/data`: Persistent user data.
- Docker named volumes: Nextcloud application files, configuration and database storage.

The actual deployment can use different directory layouts. The important thing is to keep persistent data outside the lifecycle of disposable containers.

---

# ⚙️ 5. Choose Your Deployment Profile

Before creating the Compose file, decide which configuration suits your server.

| Feature | Low-resource | Standard |
|---|---|---|
| Database | SQLite | PostgreSQL / MariaDB |
| Redis | Not included | Included |
| Background jobs | Simplified setup | Dedicated cron container |
| RAM requirements | Very limited | More comfortable resources |
| Complexity | Lower | Higher |
| Recommended use | Personal testing / small instance | Regular personal or multi-user use |

**My recommendation:** If your VM has enough memory, use PostgreSQL or MariaDB with Redis and a dedicated cron container.

SQLite is useful when resources are extremely limited, but it isn't the configuration I'd choose for a larger or more heavily used Nextcloud instance.

The official Nextcloud Docker image supports SQLite, MySQL/MariaDB and PostgreSQL.

---

# 🪶 6A. Low-Resource Deployment (SQLite)

This is the configuration I'm currently using on my Oracle Cloud Free Tier VM.

It avoids running separate database and Redis containers, reducing the number of services competing for memory.

### Environment configuration

Create the environment file:

```bash
cd /opt/nextcloud
nano .env
```

Add:

```env
NEXTCLOUD_ADMIN_USER=cloudadmin
NEXTCLOUD_ADMIN_PASSWORD=CHANGE_THIS_TO_A_STRONG_PASSWORD
NEXTCLOUD_TRUSTED_DOMAINS=your-domain.duckdns.org

SQLITE_DATABASE=nextcloud

PHP_MEMORY_LIMIT=256M
PHP_UPLOAD_LIMIT=512M
PHP_OPCACHE_MEMORY_CONSUMPTION=64
APACHE_BODY_LIMIT=536870912
```

Protect the file:

```bash
chmod 600 .env
```

Add it to `.gitignore`:

```gitignore
.env
```

Never commit passwords, private keys or other credentials to GitHub.

### Apache memory configuration

Create:

```bash
nano /opt/nextcloud/apache-memory.conf
```

Add:

```apache
<IfModule mpm_prefork_module>
    ServerLimit 3
    MaxRequestWorkers 3
    MaxConnectionsPerChild 500
</IfModule>

KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 2
```

This limits Apache to three simultaneous request workers.

These settings are intentionally conservative. They are not necessary on a server with plenty of RAM.

### Docker Compose configuration

Create:

```bash
nano /opt/nextcloud/compose.yaml
```

Add:

```yaml
services:
  app:
    image: nextcloud:apache
    container_name: nextcloud-app
    restart: unless-stopped
    init: true

    ports:
      - "127.0.0.1:8080:80"

    environment:
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: ${NEXTCLOUD_TRUSTED_DOMAINS}
      SQLITE_DATABASE: ${SQLITE_DATABASE}

      PHP_MEMORY_LIMIT: ${PHP_MEMORY_LIMIT}
      PHP_UPLOAD_LIMIT: ${PHP_UPLOAD_LIMIT}
      PHP_OPCACHE_MEMORY_CONSUMPTION: ${PHP_OPCACHE_MEMORY_CONSUMPTION}
      APACHE_BODY_LIMIT: ${APACHE_BODY_LIMIT}

    volumes:
      - nextcloud:/var/www/html
      - /srv/nextcloud/data:/var/www/html/data
      - /opt/nextcloud/apache-memory.conf:/etc/apache2/conf-enabled/zzz-memory.conf:ro

volumes:
  nextcloud:
```

Start the application:

```bash
sudo docker compose up -d
```

Check the container:

```bash
sudo docker ps
```

View initialization logs:

```bash
sudo docker logs -f nextcloud-app
```

Wait for the initial installation to complete before accessing the application.

**A note about SQLite:** This configuration is suitable for a small personal instance, but SQLite has limitations around concurrent database operations. If the server grows, migrate to PostgreSQL or MariaDB rather than simply adding more users to this configuration.

---

# 🏢 6B. Standard Deployment (PostgreSQL + Redis + Cron)

For a server with sufficient memory, this is the configuration I'd recommend.

It includes:

- PostgreSQL for persistent database storage.
- Redis for transactional file locking and memory caching.
- A dedicated cron container for background jobs.
- Nextcloud Apache as the application server.

The layout follows the official Nextcloud Docker Compose examples.

### Environment configuration

Create:

```bash
nano /opt/nextcloud/.env
```

Example:

```env
NEXTCLOUD_ADMIN_USER=cloudadmin
NEXTCLOUD_ADMIN_PASSWORD=CHANGE_THIS_TO_A_STRONG_PASSWORD
NEXTCLOUD_TRUSTED_DOMAINS=your-domain.duckdns.org

POSTGRES_DB=nextcloud
POSTGRES_USER=nextcloud
POSTGRES_PASSWORD=CHANGE_THIS_DATABASE_PASSWORD
```

Protect the file:

```bash
chmod 600 /opt/nextcloud/.env
```

Add `.env` to `.gitignore`.

For more sensitive deployments, consider Docker secrets and the supported `_FILE` environment variables rather than storing passwords directly in environment variables.

### Docker Compose configuration

Create:

```bash
nano /opt/nextcloud/compose.yaml
```

Add:

```yaml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped

    volumes:
      - db:/var/lib/postgresql/data

    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped

    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    image: nextcloud:apache
    container_name: nextcloud-app
    restart: unless-stopped
    init: true

    ports:
      - "127.0.0.1:8080:80"

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: ${NEXTCLOUD_TRUSTED_DOMAINS}

      REDIS_HOST: redis

    volumes:
      - nextcloud:/var/www/html

  cron:
    image: nextcloud:apache
    restart: unless-stopped

    entrypoint: /cron.sh

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

      REDIS_HOST: redis

    volumes:
      - nextcloud:/var/www/html

volumes:
  db:
  nextcloud:
```

Start the stack:

```bash
sudo docker compose up -d
```

Check the services:

```bash
sudo docker compose ps
```

View logs:

```bash
sudo docker compose logs -f
```

### Why PostgreSQL, Redis and Cron?

**PostgreSQL**

A proper relational database for a regular Nextcloud deployment. It stores application metadata, users, file information and other database records.

**Redis**

Redis provides caching and transactional file locking. This helps avoid database-backed locking and improves the overall application setup.

**Cron**

Nextcloud performs background jobs such as cleanup, notifications and maintenance tasks.

The dedicated cron container runs the official `/cron.sh` entrypoint. It shares the same Nextcloud volume as the application container.

This is important: the application and cron containers must use matching Nextcloud volumes.

### MariaDB alternative

You can also use MariaDB instead of PostgreSQL.

The official Nextcloud Docker examples use `mariadb:lts` with the following database settings:

```yaml
db:
  image: mariadb:lts
  restart: unless-stopped

  command: --transaction-isolation=READ-COMMITTED

  volumes:
    - db:/var/lib/mysql

  environment:
    MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    MYSQL_DATABASE: nextcloud
    MYSQL_USER: nextcloud
    MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

For Nextcloud, configure the application with:

```yaml
environment:
  MYSQL_HOST: db
  MYSQL_DATABASE: nextcloud
  MYSQL_USER: nextcloud
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

Choose PostgreSQL or MariaDB, not both. Only configure one database backend.

---

# 🔒 7. Install and Configure Caddy

Caddy acts as the reverse proxy and handles HTTPS certificates automatically.

I installed Caddy directly on the Ubuntu host rather than running it in Docker.

Installation instructions:

[Caddy Official Installation Guide](https://caddyserver.com/docs/install)

Edit the configuration:

```bash
sudo nano /etc/caddy/Caddyfile
```

Add:

```caddy
your-domain.duckdns.org {
    encode zstd gzip

    redir /.well-known/carddav /remote.php/dav/ 301
    redir /.well-known/caldav /remote.php/dav/ 301

    request_body {
        max_size 512MB
    }

    reverse_proxy 127.0.0.1:8080
}
```

Validate the configuration:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Format it:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
```

Enable Caddy:

```bash
sudo systemctl enable --now caddy
```

Reload after changes:

```bash
sudo systemctl reload caddy
```

Caddy automatically requests and renews Let's Encrypt certificates when the domain resolves correctly and the server is reachable on ports 80 and 443.

### Reverse proxy configuration

Nextcloud needs to know which reverse proxy it trusts.

The official Docker image supports configuration through `TRUSTED_PROXIES` and related environment variables.

For a host-installed reverse proxy, identify the actual source IP address seen by the Nextcloud container and configure it as a trusted proxy in Nextcloud.

Do not blindly trust every private network address. Only trusted proxy addresses should be configured.

Also, ensure that the forwarded host and HTTPS protocol are correctly recognized by Nextcloud. Otherwise, you may encounter redirect or mixed-content issues.

Reference:

[Nextcloud Reverse Proxy Configuration](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/reverse_proxy_configuration.html)

---

# 🔥 8. Firewall Configuration

There are two separate firewall layers when hosting a service on Oracle Cloud.

1. Oracle Cloud Security Lists / Network Security Groups.
2. Ubuntu's own firewall.

Both must allow the required traffic.

## Oracle Cloud Ingress Rules

In the Security List or Network Security Group associated with the VM, allow:

| Protocol | Port | Purpose |
|---|---:|---|
| TCP | 22 | SSH |
| TCP | 80 | HTTP / Let's Encrypt |
| TCP | 443 | HTTPS |

Where practical, restrict SSH access to trusted IP addresses.

## UFW Configuration

Install UFW if necessary:

```bash
sudo apt install ufw
```

Configure:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Enable:

```bash
sudo ufw enable
```

Check:

```bash
sudo ufw status verbose
```

## ⚠️ The iptables Issue I Encountered

This was probably the most frustrating part of the deployment.

Even though UFW was configured to allow incoming connections on ports 80 and 443, external connections were still being rejected.

I initially suspected a firewall issue, and that turned out to be correct.

After inspecting the INPUT chain:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

I found an early blanket REJECT rule:

```text
REJECT all -- reject-with icmp-host-prohibited
```

The problem was its position.

It appeared before the UFW chain jumps, meaning incoming traffic was rejected before UFW could evaluate its ALLOW rules.

The simplified rule order looked like this:

```text
INPUT
 |
 +-- ACCEPT established connections
 |
 +-- ACCEPT loopback
 |
 +-- ACCEPT SSH
 |
 +-- REJECT ALL  <-- Problem!
 |
 +-- UFW rules
       |
       +-- ALLOW SSH
       +-- ALLOW HTTP
       +-- ALLOW HTTPS
```

The UFW rules existed, but they were never reached for the incoming connections being rejected.

### Temporary fix

I temporarily inserted explicit ACCEPT rules before the REJECT rule:

```bash
sudo iptables -I INPUT 5 -p tcp --dport 80 \
  -m conntrack --ctstate NEW -j ACCEPT

sudo iptables -I INPUT 5 -p tcp --dport 443 \
  -m conntrack --ctstate NEW -j ACCEPT
```

This restored incoming connectivity.

### Permanent fix

A cleaner approach is to identify the source of the conflicting REJECT rule and remove or correct it in the persistent firewall configuration.

For example, inspect the current rules:

```bash
sudo iptables -L INPUT -n -v --line-numbers
sudo iptables -S
```

Look for persistent firewall configurations and services that may be restoring the conflicting rule during boot.

Do not blindly flush iptables or nftables rules.

**Important:** UFW is a firewall management frontend. It uses the underlying Linux firewall framework; it does not replace iptables/nftables.

Docker also relies on firewall rules for networking and NAT. Removing or flushing these rules can break container networking.

The temporary `iptables -I` rules are not guaranteed to survive a reboot. Correct the persistent source of the conflicting rule rather than relying on temporary runtime changes.

---

# 🌍 9. Accessing Nextcloud

Once DNS, firewall rules, Caddy and Docker are configured correctly, open:

```text
https://your-domain.duckdns.org
```

Log in using the administrator credentials configured in the `.env` file.

You should now have your own self-hosted cloud storage instance!

---

# 💾 10. Persistent Storage and Backups

Persistent storage is essential.

Deleting or recreating a container should not delete your actual Nextcloud data.

The deployment uses Docker named volumes and a host-mounted data directory.

### Low-resource deployment

| Location | Purpose |
|---|---|
| `nextcloud` Docker volume | Nextcloud application and configuration |
| `/srv/nextcloud/data` | User files |
| SQLite database | Database stored within Nextcloud's persistent application storage |

### Standard deployment

| Location | Purpose |
|---|---|
| `nextcloud` Docker volume | Nextcloud application and configuration |
| `db` Docker volume | PostgreSQL database |
| Redis container | Cache and locking |
| Cron container | Background jobs |

Back up both the Nextcloud files and the database.

For PostgreSQL, use a database dump or a consistent database backup procedure. For SQLite, ensure the database is backed up consistently.

Ideally, store backups somewhere outside the Oracle VM. Otherwise, losing the VM or its boot volume could mean losing both the application and its backups.

---

# 🔄 11. Updating Nextcloud

With Docker Compose, updating the containers is straightforward:

```bash
cd /opt/nextcloud

sudo docker compose pull
sudo docker compose up -d
```

However, always take a backup before updating.

A few things to remember:

- Check the release notes before upgrading.
- Follow Nextcloud's supported upgrade path.
- Only upgrade one major version at a time.
- Keep your database and Nextcloud volumes persistent.
- Don't use `docker compose down -v` unless you intentionally want to remove persistent volumes.

For production deployments, pin image versions rather than blindly following a moving tag. Test updates before applying them to an important instance.

Reference:

[Nextcloud Docker – Updating](https://github.com/nextcloud/docker#update-to-a-newer-version)

---

# 🛠️ 12. Useful Commands

| Task | Command |
|---|---|
| Check containers | `sudo docker compose ps` |
| View logs | `sudo docker compose logs -f` |
| Start services | `sudo docker compose up -d` |
| Stop services | `sudo docker compose stop` |
| Restart services | `sudo docker compose restart` |
| Pull updated images | `sudo docker compose pull` |
| Check Caddy status | `sudo systemctl status caddy` |
| View Caddy logs | `sudo journalctl -u caddy -f` |
| Check UFW | `sudo ufw status verbose` |
| Inspect iptables | `sudo iptables -L INPUT -n -v --line-numbers` |
| Check DNS | `nslookup your-domain.duckdns.org` |

Run Docker Compose commands from `/opt/nextcloud`.

---

# ⚠️ 13. Things to Keep in Mind

This is a personal self-hosted deployment, not a high-availability production environment.

- Around 1 GB RAM is restrictive for Nextcloud.
- SQLite is fine for a small personal deployment, but PostgreSQL or MariaDB is preferable as usage grows.
- Redis is useful for caching and transactional file locking.
- Configure background jobs using cron rather than relying on AJAX for a regular deployment.
- Keep Ubuntu, Docker, Nextcloud and Caddy updated.
- Use HTTPS whenever exposing Nextcloud to the internet.
- Never expose the Nextcloud container directly to the public internet when using a reverse proxy.
- Keep backups somewhere separate from the VM.
- Never publish credentials, private keys or `.env` files.

---

# 📚 14. Useful Resources

- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)
- [Nextcloud Docker Official Repository](https://github.com/nextcloud/docker)
- [Nextcloud Docker Compose Examples](https://github.com/nextcloud/docker/tree/master/.examples)
- [Nextcloud Administration Manual](https://docs.nextcloud.com/server/stable/admin_manual/)
- [Nextcloud System Requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html)
- [Docker Engine Documentation](https://docs.docker.com/engine/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [DuckDNS](https://www.duckdns.org/)
- [Let's Encrypt](https://letsencrypt.org/)

---

# 🎯 Final Thoughts

This started as a small self-hosting experiment using Oracle Cloud Free Tier.

I wanted to explore Docker, Linux networking, reverse proxies, DNS, SSL certificates and firewall configuration while building something I could actually use.

The Nextcloud deployment was relatively straightforward. The unexpected iptables rule was the real headache!

But that's part of self-hosting. Sometimes you end up learning more about Linux networking than the application you originally wanted to deploy. 😂

**Built and hosted using Oracle Cloud Free Tier.**
