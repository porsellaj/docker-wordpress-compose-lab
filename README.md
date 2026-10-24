# Docker WordPress Compose Lab

Academic lab. The exercise deploys WordPress and MariaDB on an Ubuntu VPS using Docker, first with individual `docker run` commands and then with Docker Compose.

> **Scope:** This is a learning lab, not a production-ready WordPress deployment. The manual section reproduces the basic architecture from the course exercise. The included Compose file is a reproducible reconstruction based on the report; the report's screenshot does not show the complete original YAML. The reconstructed file adds a dedicated database user, explicit image tags, a database health check, named volumes, and environment-variable-based credentials.

## What this lab covers

- Installing Docker on an Ubuntu VPS and checking that the engine runs.
- Running WordPress and MariaDB in separate containers.
- Connecting containers over a Docker network.
- Persisting application and database data in volumes.
- Replacing manually managed containers with a `compose.yaml` definition.
- Checking container status and logs and validating that the application responds over HTTP.

## Architecture

```text
Browser
   |
   | HTTP (port 8080 by default in the Compose example)
   v
WordPress container
   |
   | Docker network: app_network
   v
MariaDB container
   |
   +-- named volume: db_data

WordPress files -- named volume: wordpress_data
```

## Requirements

- An Ubuntu VPS or virtual machine with SSH access.
- A user with `sudo` privileges.
- Inbound TCP access to the port selected for WordPress. For a cloud VPS, limit the firewall/security-group rule to your own IP address while testing.
- Docker Engine and the Docker Compose plugin.

The original course report was produced using an Ubuntu VPS environment. For a new deployment, use a currently supported Ubuntu release rather than recreating the exercise on an unsupported operating system.

## Part 1 - Run the containers manually

This part follows the manual deployment approach described in the course report. For repeatability, it uses an explicit Docker network instead of the legacy `--link` option shown in the report. **It also preserves the report's use of the MariaDB root account for the WordPress connection only to demonstrate the original learning flow; do not use that database-account pattern in a real deployment.**

### 1. Connect to the VPS and install Docker

Connect using your SSH key, then run:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_SERVER_IP
sudo apt-get update
command -v curl || sudo apt-get install -y curl
curl -fsSL https://get.docker.com/ | sudo sh
sudo docker run --rm hello-world
```

Replace the key path and server IP with your own values. The convenience script is used here because it matches the course exercise; for managed or production systems, follow Docker's official installation instructions for your Ubuntu release.

### 2. Create a working directory and network

```bash
sudo mkdir -p /opt/wordpress-manual/database /opt/wordpress-manual/html
sudo chown -R "$USER":"$USER" /opt/wordpress-manual
cd /opt/wordpress-manual
sudo docker network create wordpress-manual-net
```

### 3. Start MariaDB

Enter a password interactively so it is not written directly into the command history:

```bash
read -rsp "MariaDB root password: " DB_ROOT_PASSWORD
printf '\n'
sudo docker run -d \
  --name wordpressdb \
  --network wordpress-manual-net \
  --restart unless-stopped \
  -e MYSQL_ROOT_PASSWORD="$DB_ROOT_PASSWORD" \
  -e MYSQL_DATABASE=wordpress \
  -v "$PWD/database:/var/lib/mysql" \
  mariadb:10.11
```

Check the container and its startup logs:

```bash
sudo docker ps
sudo docker logs wordpressdb
```

Wait for MariaDB to finish initializing before starting WordPress.

### 4. Start WordPress

The manual flow in the original report connected WordPress as the database root user. The following command reflects that training setup; use the Compose example in Part 2 for a separate, least-privilege database user.

```bash
sudo docker run -d \
  --name wordpress \
  --network wordpress-manual-net \
  --restart unless-stopped \
  -e WORDPRESS_DB_HOST=wordpressdb:3306 \
  -e WORDPRESS_DB_USER=root \
  -e WORDPRESS_DB_PASSWORD="$DB_ROOT_PASSWORD" \
  -e WORDPRESS_DB_NAME=wordpress \
  -p 80:80 \
  -v "$PWD/html:/var/www/html" \
  wordpress:php8.3-apache
```

Validate the containers and check the WordPress logs:

```bash
sudo docker ps
sudo docker logs --tail=50 wordpress
```

Open `http://YOUR_SERVER_IP/` in a browser and complete the WordPress setup wizard. Ensure that the VPS firewall/security group allows TCP port 80 from your testing IP. This manual section uses HTTP only; it does not configure TLS.

To stop and remove the manual containers before continuing to Part 2:

```bash
sudo docker stop wordpress wordpressdb
sudo docker rm wordpress wordpressdb
sudo docker network rm wordpress-manual-net
unset DB_ROOT_PASSWORD
```

The database and WordPress files from this part are stored in `/opt/wordpress-manual/`. Removing the containers does not delete those bind-mounted files.

## Part 2 - Deploy the stack with Docker Compose

Docker Compose defines the services, environment variables, network, and persistent volumes in one YAML file. This repository includes a reconstructed `docker-compose.yml` suitable for repeating the exercise without committing real credentials.

### 1. Prepare the environment file

From the repository directory:

```bash
cp .env.example .env
nano .env
chmod 600 .env
git status --short
```

Replace the example values with distinct, strong passwords. Do not commit `.env`; it is excluded by `.gitignore`. The `docker-compose.yml` file reads these values through Compose variable interpolation.

You can generate random hexadecimal values on the VPS with:

```bash
openssl rand -hex 24
```

Generate a different value for the database root password and the WordPress database user's password.

### 2. Review and validate the Compose configuration

```bash
cat docker-compose.yml
docker compose config
```

`docker compose config` renders the resolved configuration and helps catch missing environment variables or YAML/configuration errors. Avoid sharing its output if it contains secrets.

### 3. Start the services

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=50
```

The database health check allows Compose to wait for MariaDB to initialize before starting WordPress. The sample exposes WordPress on port `8080` by default to reduce the chance of conflicting with another web server.

Open `http://YOUR_SERVER_IP:8080/` and complete the WordPress setup wizard. Allow TCP port 8080 through the VPS firewall/security group only from the address you need for testing. To use port 80 instead, set `WORDPRESS_PORT=80` in `.env` and recreate the stack.

### 4. Useful management commands

```bash
# Show running services
docker compose ps

# Follow WordPress logs
docker compose logs -f wordpress

# Follow database logs
docker compose logs -f wordpressdb

# Restart the stack
docker compose restart

# Stop and remove containers and the default network (keeps named volumes)
docker compose down
```

**Data warning:** `docker compose down -v` also deletes the named volumes created by this project and permanently removes the local database and WordPress data stored in those volumes. Use it only when you intentionally want to reset the lab.

## Security notes and limitations

- The original manual exercise uses the database root account for WordPress. The Compose reconstruction instead creates and uses a dedicated database user.
- Credentials belong in the local `.env` file, not in source control. The committed `.env.example` contains placeholders only.
- The Compose example uses explicit image tags rather than `latest`, but tags are not a substitute for a planned update and patching process.
- The application is exposed over HTTP for this lab. Do not use this configuration as-is for a public production site; production deployment needs HTTPS, appropriate secrets management, backups, patching, monitoring, and a reviewed network/firewall configuration.
- Docker daemon access is highly privileged. Grant access only to trusted administrators.
- This is an academic deployment exercise, not a vulnerability assessment or a hardened server build.

## Skills demonstrated

**Linux administration · Docker · Docker Compose · MariaDB · WordPress · Container networking · Persistent storage · Basic service validation**

## References

- [OVHcloud - Install WordPress with Docker on a VPS](https://docs.ovhcloud.com/en/guides/bare-metal-cloud/virtual-private-servers/install-wordpress-docker-on-vps)
- [Docker documentation](https://docs.docker.com/)
- [Docker Compose documentation](https://docs.docker.com/compose/)
- [MariaDB container image documentation](https://hub.docker.com/_/mariadb)
- [WordPress container image documentation](https://hub.docker.com/_/wordpress)

---

