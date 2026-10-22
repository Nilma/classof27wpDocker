# WordPress + MariaDB + phpMyAdmin with Docker Compose

A step-by-step beginner project for running a WordPress website, a MariaDB database, and phpMyAdmin in separate Docker containers.

## 1. What you will build

| Service | What it does | Where to access it |
| --- | --- | --- |
| WordPress | Website and administration dashboard | http://localhost:8000 |
| MariaDB | Stores WordPress data | `db:3306` inside Docker only |
| phpMyAdmin | Browser-based database administration | http://localhost:8080 |

**How the services connect:**

```text
Your browser
  |-- localhost:8000 --> WordPress container --+
  |                                            |
  |-- localhost:8080 --> phpMyAdmin container -+--> MariaDB container (db:3306)
```

Docker Compose creates a shared network so WordPress and phpMyAdmin can reach MariaDB by the service name `db`. You do not need to expose MariaDB's port to your Mac.

## 2. Prerequisites

- Docker Desktop installed and **running** (Mac, Windows, or Linux with Docker Engine and Compose).
- Terminal access and a text editor (for example, VS Code).
- This example uses `linux/arm64`, suitable for Apple Silicon Macs. On other systems, remove the `platform:` lines to let Docker choose a compatible image.

Check Docker:

```bash
docker --version
docker compose version
docker info
```

If `docker info` cannot connect to the Docker daemon, start Docker Desktop and try again.

## 3. Create the project folder

In Terminal:

```bash
mkdir wordpress-docker
cd wordpress-docker
touch compose.yaml .env .gitignore
```

The folder will look like this:

```text
wordpress-docker/
├── compose.yaml
├── .env
├── .gitignore
└── wp-content/          # WordPress themes, plugins, uploads
```

## 4. Create the `.env` file

Open `.env` and paste:

```dotenv
MYSQL_ROOT_PASSWORD='MyR00tMySQLPa$5w0rD'
MYSQL_DATABASE=MyWordPressDatabaseName
MYSQL_USER=MyWordpressUser
MYSQL_PASSWORD='Pa$5w0rD'

WORDPRESS_PORT=8000
PHPMYADMIN_PORT=8080
```

**Important:** Single quotes in `.env` preserve the literal `$` characters. These are **example passwords for local learning**: change them for your own project. Do not upload `.env` to GitHub.

The database account (`MYSQL_USER`) is **not** your WordPress website administrator account; you will create that later in the browser.

## 5. Create the `compose.yaml` file

Open `compose.yaml` and paste:

```yaml
services:
  db:
    image: mariadb:latest
    platform: linux/arm64
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    platform: linux/arm64
    restart: always
    depends_on:
      - db
    ports:
      - "${WORDPRESS_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
    volumes:
      - ./wp-content/:/var/www/html/wp-content

  phpmyadmin:
    image: phpmyadmin:latest
    platform: linux/arm64
    restart: always
    depends_on:
      - db
    ports:
      - "${PHPMYADMIN_PORT}:80"
    environment:
      PMA_HOST: db

volumes:
  db_data:
```

**Understanding the file:**

- `services:` defines three containers: `db`, `wordpress`, and `phpmyadmin`.
- `image:` selects a prebuilt Docker image; no Dockerfile is required.
- `environment:` passes settings into each container.
- `${MYSQL_USER}` and similar placeholders are replaced using `.env`.
- `ports: "8000:80"` means **Mac port 8000 → container port 80**.
- `depends_on:` controls start order, **not** database readiness; WordPress may need a few moments while MariaDB initializes.
- `db_data` is a **named volume** that persists database data.
- `./wp-content/` is a **bind mount** mapping a local folder into WordPress for plugins, themes, and uploads.
- `restart: always` restarts containers after Docker restarts, unless stopped manually.

> For reproducible or production deployments, pin image versions rather than using `latest`, and use proper secret management. This configuration is intended for local learning.

## 6. Keep passwords out of Git

Add this to `.gitignore`:

```gitignore
.env
wp-content/
```

Optionally create a shareable `.env.example` with placeholders (not real passwords):

```dotenv
MYSQL_ROOT_PASSWORD='replace-with-root-password'
MYSQL_DATABASE=MyWordPressDatabaseName
MYSQL_USER=MyWordpressUser
MYSQL_PASSWORD='replace-with-user-password'
WORDPRESS_PORT=8000
PHPMYADMIN_PORT=8080
```

A classmate can copy `.env.example` to `.env` and supply their own passwords.

## 7. Validate and start the containers

Run these commands **from the folder containing `compose.yaml` and `.env`**:

```bash
docker compose config --quiet
docker compose up -d
```

- `config --quiet` checks whether Compose can parse the configuration.
- `up` creates and starts the services.
- `-d` runs them in the background.
- The first run downloads the images and initializes MariaDB; allow a minute or two.

Check their status:

```bash
docker compose ps
```

You should see the `db`, `wordpress`, and `phpmyadmin` services. To inspect logs:

```bash
docker compose logs -f
```

Press **Ctrl+C** to stop viewing logs (the containers keep running).

## 8. Install WordPress in the browser

1. Open **http://localhost:8000**.
2. Select a language.
3. Enter a site title.
4. **Create a WordPress administrator username and password** and provide an email address.
5. Click **Install WordPress**.
6. Log in to your website's dashboard at **http://localhost:8000/wp-admin**.

**Remember:** The WordPress admin login is **different** from the MariaDB login in `.env`.

## 9. Open phpMyAdmin and inspect the database

1. Open **http://localhost:8080**.
2. Log in with the database credentials from `.env`:
   - **Username:** `MyWordpressUser`
   - **Password:** `Pa$5w0rD` (if you used the sample `.env` exactly)
3. Select **MyWordPressDatabaseName** on the left.
4. Browse tables such as `wp_posts`, `wp_options`, and `wp_users` (the prefix may differ).

To find the WordPress login username, browse the `wp_users` table and look at the `user_login` column. **Do not confuse the database account with the WordPress account.** Avoid editing WordPress tables directly unless you understand the consequences.

## 10. Everyday Docker Compose commands

Run commands from the project directory:

| Command | Purpose |
| --- | --- |
| `docker compose up -d` | Create/start services in the background |
| `docker compose ps` | Show container status |
| `docker compose logs -f wordpress` | Follow WordPress logs |
| `docker compose logs -f db` | Follow database logs |
| `docker compose stop` | Stop containers without removing them |
| `docker compose start` | Start previously stopped containers |
| `docker compose down` | Stop and remove containers/network, **keep named volume** |
| `docker compose up -d --force-recreate` | Recreate containers with current configuration |
| `docker compose down -v` | **Delete containers AND named database volume** |

**Warning:** `docker compose down -v` deletes your MariaDB data. Use it only when you intentionally want a fresh database and have backed up anything important.

## 11. Troubleshooting

### WordPress does not open on port 8000

```bash
docker compose ps
docker compose logs --tail=100 wordpress
```

Confirm Docker Desktop is running and that `.env` says `WORDPRESS_PORT=8000`. If port 8000 is occupied, change it to another free port (for example, `8001`) and run `docker compose up -d` again. Then open `http://localhost:8001`.

### phpMyAdmin does not open on port 8080

Check `docker compose ps` and `docker compose logs --tail=100 phpmyadmin`. Change `PHPMYADMIN_PORT` if port 8080 is already in use.

### WordPress says “Error establishing a database connection”

MariaDB may still be starting. Wait briefly, refresh the page, and inspect:

```bash
docker compose logs --tail=100 db
```

Check that the WordPress `WORDPRESS_DB_*` values match the database variables.

### “The username MyWordpressUser is not registered on this site”

You tried to sign into **WordPress** with the **MariaDB** username. Use the WordPress admin username you created during installation. You can look up the WordPress username in phpMyAdmin → your database → `wp_users` → `user_login`.

### I changed `.env` but my database password did not change

MariaDB's initialization variables only create users/passwords when the data directory is **first initialized**. Recreating containers does not change credentials in an existing `db_data` volume. Change the account password inside MariaDB, or—**only if you can lose the data**—reset the volume with `docker compose down -v` followed by `docker compose up -d`.

### Password contains `$`

In a Compose `.env` file, use **single quotes** around a password containing `$`, as in `MYSQL_PASSWORD='Pa$5w0rD'`. In a Compose YAML value written directly, `$$` is used to escape a literal dollar sign. Avoid displaying your expanded configuration in shared screenshots because it may expose passwords.

## 12. What you learned

- **Image:** the template used to create a container (for example, `wordpress:latest`).
- **Container:** a running instance of an image.
- **Docker Compose:** starts and connects multiple services from one YAML file.
- **`.env`:** supplies configuration values used by Compose; it does **not** encrypt secrets.
- **Named volume:** preserves database data when containers are removed.
- **Bind mount:** shares a local folder with a container.
- **Port mapping:** makes a service available from your computer.


