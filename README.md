# Notes App

A simple notes application with a React frontend, a Django REST API, and a MySQL database. Docker Compose runs the backend stack locally, with Nginx as the public entry point.

## Project structure

- `mynotes/` — React frontend source and its built assets.
- `api/` — Django REST API and the `Note` model.
- `notesapp/` — Django project configuration and URL routing.
- `nginx/` — Nginx image and reverse-proxy configuration.
- `docker-compose.yml` — Services, ports, network, and database storage.

## Requirements

- Docker Desktop (Windows/macOS) or Docker Engine with the Docker Compose plugin (Linux).
- Git, if you are cloning the repository.

Check that Docker and Compose are available:

```sh
docker --version
docker compose version
```

## Run with Docker Compose

The Django service reads database settings from a `.env` file in the repository root. Create or check that file before starting Compose. Its variable names and local Compose-compatible values are:

```dotenv
DB_NAME=test_db
DB_USER=root
DB_PASSWORD=root
DB_HOST=db
DB_PORT=3306
```

These values are for local development only. The Compose file also sets the MySQL root password to `root`; use secure, separately managed credentials for any non-local deployment.

Build the images and start the services:

```sh
docker compose up --build -d
```

Compose starts three services on the private `notes-app-nw` network:

| Service | Purpose | Host access |
| --- | --- | --- |
| `nginx` | Reverse proxy forwarding HTTP traffic to Django | `http://localhost` (port 80) |
| `django_app` | Django REST API and Gunicorn application server | `http://localhost:8000` |
| `db` | MySQL database | Internal to the Compose network (port 3306) |

Open `http://localhost` to use the application. Nginx forwards requests to Django, and Django serves the built frontend and API. The React source is in `mynotes/`; Compose does not run a separate Node.js frontend service.

The Django container runs database migrations before starting Gunicorn. Compose also defines health checks for Django and MySQL. `depends_on` controls service startup order, but does not wait for a dependency to become healthy.

## Common commands

Run these from the directory containing `docker-compose.yml`:

```sh
# Start services in the background
docker compose up -d

# Rebuild images after changing a Dockerfile or backend files, then start
docker compose up --build -d

# Show service state
docker compose ps

# Follow logs from every service, or one service
docker compose logs -f
docker compose logs -f django_app

# Run a Django management command
docker compose exec django_app python manage.py check
docker compose exec django_app python manage.py createsuperuser

# Open a shell in the Django container
docker compose exec django_app sh

# Stop and remove the containers and Compose network
docker compose down
```

`docker compose down` preserves the MySQL files mounted at `data/mysql/db`, so database contents remain available when you start the stack again. Back up that directory before making any manual changes to its contents.

To validate the Compose configuration without starting containers:

```sh
docker compose config --quiet
```

## Docker Compose basics

- **Services** describe the containers Compose runs. This project declares `nginx`, `django_app`, and `db`.
- **Build and image** tell Compose how to create images. Nginx is built from `nginx/`; Django is built from the repository root using `Dockerfile`; MySQL uses the upstream `mysql` image.
- **Ports** publish container ports on the host. For example, `80:80` means port 80 on your computer forwards to port 80 in the Nginx container.
- **Service discovery and networks** let containers on `notes-app-nw` find each other by service name. Django connects to MySQL at host `db`; Nginx proxies requests to Django.
- **Environment variables** configure the Django database connection. Compose loads them from `.env` using `env_file`.
- **Bind mounts** connect a host path to a container path. `./data/mysql/db:/var/lib/mysql` keeps database files in the repository's `data/mysql/db` directory.
- **Health checks** periodically test whether a container's service responds. They report health; they do not by themselves restart an unhealthy process.
- **`depends_on`** establishes startup order. It does not guarantee that a database is ready to accept connections when Django starts.

## API

The API is available at `http://localhost/api/` (or directly at `http://localhost:8000/api/`):

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/api/` | List available API routes |
| `GET` | `/api/notes/` | List notes |
| `GET` | `/api/notes/<id>/` | Get one note |
| `POST` | `/api/notes/create/` | Create a note; send JSON such as `{"body":"Remember this"}` |
| `PUT` | `/api/notes/<id>/update/` | Update a note |
| `DELETE` | `/api/notes/<id>/delete/` | Delete a note |

Django's admin is available at `http://localhost/admin/`. Create an admin account with the `createsuperuser` command above.

## Troubleshooting

- **Django exits or cannot connect to MySQL:** Check that `.env` exists at the repository root and that its database values match the MySQL settings in `docker-compose.yml`. Check `docker compose logs db django_app`.
- **The app is not reachable:** Check `docker compose ps` and `docker compose logs nginx`. Port 80 must be available; port 8000 must also be available if you want direct access to Django.
- **A code change is not reflected:** Rebuild and restart with `docker compose up --build -d`. The frontend source is not built by a Compose service; update the React build assets if changing the frontend.

