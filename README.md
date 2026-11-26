*This project has been created as part of the 42 curriculum by brnascim.*

---

# Inception

## Description

**Inception** is a system administration project that builds a small but complete web infrastructure entirely with Docker. Every service runs in its own dedicated container, built from scratch from a `debian:bookworm` base image — no pre-made images from DockerHub.

The stack delivers a fully functional WordPress site with:
- **NGINX** as the sole HTTPS entry point (TLSv1.2 / TLSv1.3 on port 443)
- **WordPress + php-fpm** for the CMS
- **MariaDB** for the relational database

### Key Design Choices

#### Virtual Machines vs Docker
| | Virtual Machine | Docker Container |
|-|-----------------|-----------------|
| Isolation | Full OS-level | Process-level (namespaces + cgroups) |
| Startup | Minutes | Seconds |
| Overhead | High (full kernel) | Low (shared host kernel) |
| Use case | Full OS isolation, legacy apps | Lightweight, reproducible services |

Docker containers share the host kernel, making them lighter and faster than VMs. They are not VMs — they don't boot a full OS.

#### Secrets vs Environment Variables
| | Secrets | Environment Variables |
|-|---------|----------------------|
| Storage | Files mounted at `/run/secrets/` | Process environment |
| Visibility | Available only inside the container | Visible in `docker inspect` |
| Use case | Passwords, API keys | Config values (domain, usernames) |

Passwords are stored in `secrets/` files and mounted as Docker secrets. Non-sensitive configuration (domain name, usernames) lives in `.env`.

#### Docker Network vs Host Network
| | Docker Bridge Network | Host Network |
|-|-----------------------|-------------|
| Isolation | Containers on a private subnet | Container shares host network |
| Security | Services unreachable from outside unless port-mapped | All container ports exposed on host |
| DNS | Containers resolve each other by service name | Must use host IPs |

This project uses a custom **bridge network** (`inception_net`). Containers communicate by service name (e.g., `wordpress:9000`, `mariadb:3306`). Only port `443` (NGINX) is published to the host.

#### Docker Volumes vs Bind Mounts
| | Docker Volumes | Bind Mounts |
|-|----------------|-------------|
| Management | Managed by Docker (`docker volume`) | Managed directly by the host OS |
| Portability | High (decoupled from host directory structure) | Tied to a specific host path |
| Isolation | Safer from accidental host modifications | Directly exposed on the host filesystem |

This project uses **named Docker volumes** configured with `driver_opts` (`type: none`, `o: bind`) to combine Docker volume management with explicit host persistence at `/home/brnascim/data/{mariadb,wordpress}`.

---

## Instructions

### Prerequisites
- Docker Engine ≥ 24
- Docker Compose v2
- `make`
- Add to `/etc/hosts`: `127.0.0.1  brnascim.42.fr`
- Create `srcs/.env` and `secrets/` as detailed in `DEV_DOC.md`

### Run

```bash
# Clone and enter the repository
git clone <repo-url> inception && cd inception

# Build and start all containers
make

# Stop without removing data
make down

# Full teardown (removes containers, volumes, images)
make fclean
```

---

## Resources

### Documentation
- [Docker Docs](https://docs.docker.com/)
- [Docker Compose reference](https://docs.docker.com/compose/compose-file/)
- [NGINX docs](https://nginx.org/en/docs/)
- [WordPress CLI (wp-cli)](https://wp-cli.org/)
- [MariaDB Knowledge Base](https://mariadb.com/kb/en/)

### AI Usage
AI was used for the following tasks:
- **Scaffolding**: generating the initial directory structure and boilerplate Dockerfiles
- **Configuration**: drafting `nginx.conf`, php-fpm pool, and MariaDB config
- **Entrypoint scripts**: structuring the init logic for MariaDB bootstrap and WordPress WP-CLI automation
- **Documentation**: drafting `README.md`, `USER_DOC.md`, and `DEV_DOC.md` structure

All AI-generated content was reviewed, tested, and adjusted. I take full responsibility for every line in this repository.