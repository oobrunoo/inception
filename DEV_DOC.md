# DEV_DOC.md — Developer Documentation

## Prerequisites

Install the following on your system (or inside your VM):

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y docker.io docker-compose-v2 make git

# Add your user to the docker group (avoids sudo for docker commands)
sudo usermod -aG docker $USER
newgrp docker
```

Also add the domain to `/etc/hosts`:
```bash
echo "127.0.0.1  brnascim.42.fr" | sudo tee -a /etc/hosts
```

---

## Setting Up From Scratch

### 1. Clone the repository

```bash
git clone <repo-url> inception
cd inception
```

### 2. Create the secrets files

The `secrets/` directory is gitignored. You must create these files manually:

```bash
mkdir -p secrets

# MariaDB user password
echo "YourStrongPassword1" > secrets/db_password.txt

# MariaDB root password
echo "YourStrongRootPassword1" > secrets/db_root_password.txt

# WordPress admin and regular user passwords
cat > secrets/credentials.txt <<EOF
ADMIN_PASSWORD=YourAdminPass1
USER_PASSWORD=YourUserPass1
EOF
```

> ⚠️ Never commit these files. They are in `.gitignore`.

### 3. Create the environment file

Non-sensitive configuration lives in `srcs/.env` (also gitignored):

```dotenv
DOMAIN_NAME=brnascim.42.fr
MYSQL_DATABASE=wordpress_db
MYSQL_USER=wp_user
WP_TITLE=Inception brnascim
WP_ADMIN_USER=superuser
WP_ADMIN_EMAIL=superuser@42.fr
WP_USER_LOGIN=editor_user
WP_USER_EMAIL=editor@42.fr
```

### 4. Host data directories

The `Makefile` creates these automatically on `make`, or you can create them manually:

```bash
sudo mkdir -p /home/brnascim/data/mariadb
sudo mkdir -p /home/brnascim/data/wordpress
```

---

## Build and Launch

```bash
# Build all images and start all containers in detached mode
make

# Equivalent manual command
docker compose -f srcs/docker-compose.yml up -d --build
```

On first boot, the WordPress setup script will:
1. Wait for MariaDB to be ready
2. Download WordPress core
3. Configure `wp-config.php`
4. Install WordPress with admin credentials
5. Create the second (author) user

This takes ~30–60 seconds on first run.

---

## Useful Container Management Commands

```bash
# Follow logs for all services
make logs

# Follow logs for a single service
docker logs -f wordpress

# Open an interactive shell in a container
docker exec -it wordpress bash
docker exec -it mariadb bash

# Run a WP-CLI command inside the WordPress container
docker exec wordpress wp user list --allow-root

# Check MariaDB from inside the container
docker exec mariadb mysql -u root -p -e "SHOW DATABASES;"

# Inspect the bridge network
docker network inspect srcs_inception_net

# List all images built for this project
docker images | grep -E "nginx|wordpress|mariadb"
```

---

## Project Data Location

All persistent data is stored on the **host** filesystem:

| Data | Host path | Volume name |
|------|-----------|-------------|
| MariaDB database files | `/home/brnascim/data/mariadb` | `wordpress_db` |
| WordPress site files | `/home/brnascim/data/wordpress` | `wordpress_files` |

Data **survives** `make down` (stop only).  
Data is **removed** by `make clean` or `make fclean`.

---

## Project Structure

```
inception/
├── Makefile                        # Build orchestration
├── README.md                       # Project overview
├── USER_DOC.md                     # End-user documentation
├── DEV_DOC.md                      # This file
├── .gitignore                      # Ignores secrets/ and .env
├── secrets/                        # Secret files (gitignored)
│   ├── db_password.txt
│   ├── db_root_password.txt
│   └── credentials.txt
└── srcs/
    ├── .env                        # Non-sensitive env vars (gitignored)
    ├── docker-compose.yml          # Service definitions
    └── requirements/
        ├── nginx/                  # NGINX (TLS termination on port 443)
        │   ├── Dockerfile
        │   ├── conf/nginx.conf
        │   └── tools/generate_ssl.sh
        ├── wordpress/              # WordPress + php-fpm
        │   ├── Dockerfile
        │   ├── conf/www.conf
        │   └── tools/wp_setup.sh
        └── mariadb/                # MariaDB
            ├── Dockerfile
            ├── conf/my.cnf
            └── tools/db_init.sh
```

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| `https://brnascim.42.fr` unreachable | `/etc/hosts` entry missing | `echo "127.0.0.1 brnascim.42.fr" \| sudo tee -a /etc/hosts` |
| WordPress shows DB connection error | MariaDB not ready yet | Wait ~30s and refresh, or check `docker logs wordpress` |
| Port 443 already in use | Another service using port 443 | `sudo lsof -i :443` and stop the conflicting process |
| `make clean` fails on volumes | Volumes still in use | Run `make down` first, then `make clean` |