# Documentation: Hosting a Django REST Framework (DRF) Project on AWS EC2 with Docker and HTTPS

This guide explains how to deploy a Django REST Framework (DRF) project on an AWS EC2 instance using Docker, PostgreSQL, Redis, Celery, Celery Beat, and Nginx, secured with HTTPS via Let’s Encrypt. It covers setting up the environment, generating migrations, configuring Nginx, enabling HTTPS, and verifying the deployment. The guide also includes troubleshooting steps for common issues, making it accessible to anyone with basic technical knowledge.

## Prerequisites

Before starting, ensure you have:

- **AWS Account**: Access to the AWS Management Console.
- **Domain Name**: A registered domain (e.g., `yourdomainname`) for HTTPS setup.
- **GitHub Repository**: Your DRF project hosted on GitHub, including a `Dockerfile`, `requirements.txt`, and Django project files.
- **SSH Client**: Tools like PuTTY (Windows) or Terminal (Linux/macOS) for SSH access.
- **Basic Knowledge**: Familiarity with Linux commands, Docker, and Django.

## Project Overview

- **Application**: A DRF backend with Django’s `AbstractUser` for authentication.
- **Components**:
  - **Web**: Django app served via Daphne (ASGI).
  - **Database**: PostgreSQL for persistent data.
  - **Redis**: For Celery task queue and caching.
  - **Celery**: For background tasks.
  - **Celery Beat**: For scheduled tasks.
  - **Nginx**: Reverse proxy for HTTP/HTTPS and static file serving.
- **Domain**:
  - Frontend (if applicable): `yourdomainname`.
  - Backend: `api.yourdomainname`.
- **EC2 Instance**: Ubuntu with public IP (e.g., `13.127.106.180`).
- **HTTPS**: Secured with Let’s Encrypt SSL certificates.

## Configuration Files

Below are the final, working configuration files used in this deployment.

### `docker-compose.yml`

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    command: >
      sh -c "python manage.py collectstatic --noinput &&
             daphne -b 0.0.0.0 -p 8000 yourprojectname.asgi:application"
    volumes:
      - .:/app
      - ./staticfiles:/app/staticfiles
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db
      - redis
    networks:
      - yourprojectname-network

  db:
    image: postgres:16.4
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - yourprojectname-network

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - yourprojectname-network

  celery:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A yourprojectname worker --loglevel=info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - web
      - redis
    networks:
      - yourprojectname-network

  celery-beat:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A yourprojectname beat --loglevel=info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - web
      - redis
    networks:
      - yourprojectname-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./staticfiles:/app/staticfiles
      - /etc/letsencrypt:/etc/letsencrypt
    depends_on:
      - web
    networks:
      - yourprojectname-network

volumes:
  postgres_data:
  redis_data:

networks:
  yourprojectname-network:
    driver: bridge
```

### `Dockerfile`

```dockerfile
FROM python:3.13-slim

# Create appuser and app directory
RUN useradd -m -u 1000 appuser && mkdir /app && chown -R appuser:appuser /app

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y gcc libpq-dev curl && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt

RUN mkdir -p /app/staticfiles && chmod -R 777 /app/staticfiles

# Copy project files
COPY . .

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Use non-root user
USER appuser

EXPOSE 8000

# For Daphne container (in docker-compose) we override CMD anyway
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "3", "yourprojectname.wsgi:application"]
```

### `nginx/nginx.conf`

```text
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name api.yourdomainname 13.127.106.180;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.yourdomainname;

    ssl_certificate /etc/letsencrypt/live/api.yourdomainname/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourdomainname/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    client_max_body_size 100M;

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /app/staticfiles/;
    }

    location /ws/ {
        proxy_pass http://django;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```


## Step-by-Step Setup Guide

### 1. Launch an AWS EC2 Instance

1. **Log in to AWS Console**:

   - Go to AWS Management Console and navigate to EC2.

2. **Create an EC2 Instance**:

   - Click **Launch Instance**.
   - **Name**: `yourprojectname-server` (or your choice).
   - **OS**: Choose **Ubuntu Server 22.04 LTS** (free tier eligible).
   - **Instance Type**: Select `t2.micro` (free tier) or `t2.small` for better performance.
   - **Key Pair**: Create or use an existing key pair (e.g., `yourprojectname-key.pem`) for SSH access. Download and store securely.
   - **Network Settings**:
     - Allow **SSH** (port 22) from **My IP** (or `0.0.0.0/0` for broader access, less secure).
     - Allow **HTTP** (port 80) and **HTTPS** (port 443) from `0.0.0.0/0`.
   - **Storage**: Allocate at least 20GB for Docker images, volumes, and logs.
   - Click **Launch Instance**.

3. **Get Public IP**:

   - Once running, note the instance’s public IP (e.g., `13.127.106.180`).

4. **SSH into the Instance**:

   - Open a terminal (or PuTTY):

     ```bash
     ssh -i yourprojectname-key.pem ubuntu@13.127ॵ.106.180
     ```

   - If permission errors occur, run:

     ```bash
     chmod 400 yourprojectname-key.pem
     ```

### 2. Prepare the EC2 Environment

1. **Update the System**:

   - Update packages and resolve any restart requirements:

     ```bash
     sudo apt update
     sudo apt upgrade -y
     sudo reboot
     ```

   - Reconnect after reboot:

     ```bash
     ssh -i yourprojectname-key.pem ubuntu@13.127.106.180
     ```

2. **Install Docker and Docker Compose**:

   - Install Docker:

     ```bash
     sudo apt install -y docker.io
     sudo systemctl start docker
     sudo systemctl enable docker
     sudo usermod -aG docker ubuntu
     ```

   - Install Docker Compose:

     ```bash
     sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.6/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
     sudo chmod +x /usr/local/bin/docker-compose
     docker-compose --version
     ```

   - Log out and back in to apply the Docker group change:

     ```bash
     exit
     ssh -i yourprojectname-key.pem ubuntu@13.127.106.180
     ```

3. **Verify Disk Space**:

   - Ensure sufficient space:

     ```bash
     df -h
     ```

   - If low, resize the EBS volume in the AWS Console (EC2 &gt; Volumes &gt; Modify Volume).

### 3. Clone and Configure the Project

1. **Clone the Repository**:

   - Clone your GitHub repo:

     ```bash
     git clone <your-github-repo-url> yourprojectname
     cd yourprojectname
     ```

   - Replace `<your-github-repo-url>` with your repository URL (e.g., `https://github.com/username/yourprojectname.git`).

2. **Verify Files**:

   - Ensure the directory contains:
     - `Dockerfile`
     - `docker-compose.yml`
     - `requirements.txt`
     - `yourprojectname/` (Django project directory)
     - Apps with models (e.g., `users/models.py` for `CustomUser`).

3. **Create Environment File**:

   - Create a `.env` file for secrets:

     ```bash
     nano .env
     ```

   - Add:

     ```text
     DB_NAME=yourprojectname
     DB_USER=yourprojectname_user
     DB_PASSWORD=your_secure_password
     REDIS_URL=redis://redis:6379/0
     SECRET_KEY=your_django_secret_key
     DEBUG=False
     ALLOWED_HOSTS=13.127.106.180,api.yourdomainname
     ```

   - Generate a secure `SECRET_KEY`:

     ```bash
     python -c "import secrets; print(secrets.token_urlsafe(50))"
     ```

   - Replace `your_secure_password` with a strong password.

4. **Configure Django Settings**:

   - Edit `yourprojectname/settings.py` to match the environment:

     ```python
     import os
     from pathlib import Path
     
     BASE_DIR = Path(__file__).resolve().parent.parent
     
     SECRET_KEY = os.getenv("SECRET_KEY")
     DEBUG = os.getenv("DEBUG", "False") == "True"
     ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS", "").split(",")
     
     INSTALLED_APPS = [
         "django.contrib.admin",
         "django.contrib.auth",
         "django.contrib.contenttypes",
         "django.contrib.sessions",
         "django.contrib.messages",
         "django.contrib.staticfiles",
         "rest_framework",
         "users",  # Your app name
     ]
     
     AUTH_USER_MODEL = "users.CustomUser"  # Replace with your app and model name
     
     DATABASES = {
         "default": {
             "ENGINE": "django.db.backends.postgresql",
             "NAME": os.getenv("DB_NAME"),
             "USER": os.getenv("DB_USER"),
             "PASSWORD": os.getenv("DB_PASSWORD"),
             "HOST": "db",
             "PORT": "5432",
         }
     }
     
     REDIS_URL = os.getenv("REDIS_URL")
     CELERY_BROKER_URL = REDIS_URL
     CELERY_RESULT_BACKEND = REDIS_URL
     
     STATIC_URL = "/static/"
     STATIC_ROOT = os.path.join(BASE_DIR, "staticfiles")
     ```

   - Ensure `AUTH_USER_MODEL` points to your `CustomUser` model (e.g., `users.CustomUser`).

   - Add `rest_framework` and your apps to `INSTALLED_APPS`.
     
### 4. Configure Nginx

Nginx serves as a reverse proxy and handles static files and HTTPS.

1. **Create Nginx Configuration**:

   - Create a directory and config file:

     ```bash
     mkdir nginx
     nano nginx/nginx.conf
     ```

   - Add the provided configuration:

     ```text
     upstream django {
         server web:8000;
     }
     
     server {
         listen 80;
         server_name api.yourdomainname 13.127.106.180;
         return 301 https://$server_name$request_uri;
     }
     
     server {
         listen 443 ssl;
         server_name api.yourdomainname;
     
         ssl_certificate /etc/letsencrypt/live/api.yourdomainname/fullchain.pem;
         ssl_certificate_key /etc/letsencrypt/live/api.yourdomainname/privkey.pem;
         ssl_protocols TLSv1.2 TLSv1.3;
         ssl_prefer_server_ciphers on;
     
         client_max_body_size 100M;
     
         location / {
             proxy_pass http://django;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     
         location /static/ {
             alias /app/staticfiles/;
         }
     
         location /ws/ {
             proxy_pass http://django;
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "upgrade";
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
         }
     }
     ```

   - This redirects HTTP to HTTPS, serves static files, and supports WebSocket connections (if used).

2. **Update** `docker-compose.yml`:

   - Use the provided `docker-compose.yml` (below) to include the Nginx service.

3. **Collect Static Files**:

   - Modify the `web` service command in `docker-compose.yml` to collect static files:

     ```yaml
     command: >
       sh -c "python manage.py collectstatic --noinput &&
              daphne -b 0.0.0.0 -p 8000 yourprojectname.asgi:application"
     ```

   - Run:

     ```bash
     docker-compose up -d
     docker-compose exec web python manage.py collectstatic --noinput
     ```

### 5. Generate and Apply Migrations

If your repository lacks migration files, generate them for the database schema.

1. **Build Docker Images**:

   ```bash
   docker-compose build
   ```

2. **Generate Migrations**:

   - Create migration files for all apps, including `CustomUser`:

     ```bash
     docker-compose run --rm web python manage.py makemigrations
     ```

   - This creates migration files in app directories (e.g., `users/migrations/`).

3. **Apply Migrations**:

   ```bash
   docker-compose run --rm web python manage.py migrate
   ```

4. **Create a Superuser** (optional, for admin access):

   ```bash
   docker-compose run --rm web python manage.py createsuperuser
   ```

   - Follow prompts to set username, email, and password.

5. **Verify Migrations**:

   - Check for migration files:

     ```bash
     ls users/migrations/
     ```

   - Expected: Files like `0001_initial.py`.

### 6. Set Up HTTPS with Let’s Encrypt

Secure the backend at `api.yourdomainname` with SSL.

1. **Configure DNS**:

   - Log in to your domain registrar (e.g., GoDaddy, Namecheap).

   - Add an A record:

     - **Name**: `api`
     - **Value**: `13.127.106.180` (your EC2 public IP)
     - **TTL**: Default or 300 seconds

   - Verify DNS propagation:

     ```bash
     ping api.yourdomainname
     ```

     - Expected: Resolves to `13.127.106.180`.

2. **Install Certbot**:

   - Install Certbot and the Nginx plugin on the EC2 instance:

     ```bash
     sudo apt install -y certbot python3-certbot-nginx
     ```

3. **Obtain SSL Certificate**:

   - Run Certbot to get certificates for `api.yourdomainname`:

     ```bash
     sudo certbot --nginx -d api.yourdomainname
     ```

   - Follow prompts:

     - Agree to terms.
     - Choose to redirect HTTP to HTTPS (recommended).

   - Certificates are stored in `/etc/letsencrypt/live/api.yourdomainname/`.

4. **Verify Nginx Configuration**:

   - Ensure `nginx.conf` references the correct certificate paths (already set in the provided config).

   - Test Nginx config:

     ```bash
     docker-compose exec nginx nginx -t
     ```

     - Expected: `syntax is ok`.

5. **Restart Services**:

   ```bash
   docker-compose down
   docker-compose up -d
   ```

### 7. Update EC2 Security Group

Ensure the EC2 instance allows necessary traffic.

1. **Go to AWS Console**:
   - Navigate to EC2 &gt; Security Groups &gt; Select your instance’s security group.
2. **Edit Inbound Rules**:
   - Add or verify:
     - **SSH**: Port 22, Source: Your IP (or `0.0.0.0/0` for broader access).
     - **HTTP**: Port 80, Source: `0.0.0.0/0`.
     - **HTTPS**: Port 443, Source: `0.0.0.0/0`.
   - Remove (if unused):
     - Port 5432 (PostgreSQL).
     - Port 6379 (Redis).
     - Port 8000 (handled by Nginx).
3. **Save Changes**.

### 8. Start the Application

1. **Run Docker Compose**:

   ```bash
   docker-compose up -d
   ```

2. **Verify Containers**:

   ```bash
   docker ps
   ```

   - Expected: Six running containers:
     - `yourprojectname_web_1`
     - `yourprojectname_db_1`
     - `yourprojectname_redis_1`
     - `yourprojectname_celery_1`
     - `yourprojectname_celery-beat_1`
     - `yourprojectname_nginx_1`

### 9. Verify the Deployment

Check that all components are working correctly.

 1. **Check Container Status**:

    ```bash
    docker ps
    ```

    - Ensure all six containers are `Up`.

 2. **View Logs**:

    ```bash
    docker-compose logs web
    docker-compose logs db
    docker-compose logs redis
    docker-compose logs celery
    docker-compose logs celery-beat
    docker-compose logs nginx
    ```

    - Look for:
      - `web`: Daphne listening on `0.0.0.0:8000`.
      - `db`: `database system is ready to accept connections`.
      - `redis`: `Ready to accept connections`.
      - `celery`: `celery@<hostname> ready`.
      - `celery-beat`: Scheduler activity.
      - `nginx`: No errors; HTTP/HTTPS requests logged.

 3. **Test Database**:

    ```bash
    docker-compose exec db psql -U yourprojectname_user -d yourprojectname
    ```

    - Run:

      ```sql
      \dt
      ```

      - Expected: Lists tables like `auth_user`, `users_customuser`.

 4. **Test Redis**:

    ```bash
    docker-compose exec redis redis-cli ping
    ```

    - Expected: `PONG`.

 5. **Test HTTP**:

    ```bash
    curl http://13.127.106.180
    ```

    - Expected: Redirects to HTTPS or returns a Django response.

 6. **Test HTTPS**:

    ```bash
    curl https://api.yourdomainname
    ```

    - Expected: Returns a Django response (e.g., DRF API root).
    - In a browser, visit `https://api.yourdomainname`:
      - Expected: Loads DRF browsable API or app root with a valid SSL certificate (padlock icon).

 7. **Test Admin Interface**:

    - Visit `https://api.yourdomainname/admin/` in a browser.
    - Log in with superuser credentials.
    - Expected: Admin dashboard loads, showing `CustomUser` and other models.

 8. **Test API Endpoints**:

    - If you have an endpoint (e.g., `/api/users/`):

      ```bash
      curl https://api.yourdomainname/api/users/
      ```

      - Expected: Returns JSON (e.g., `[]` or user data).

 9. **Test Static Files**:

    ```bash
    curl https://api.yourdomainname/static/admin/css/base.css
    ```

    - Expected: Returns CSS content.

10. **Test Celery**:

    - Open a Django shell:

      ```bash
      docker-compose exec web python manage.py shell
      ```

    - Run:

      ```python
      from celery import shared_task
      @shared_task
      def test_task():
          print("Test task ran!")
          return "Done"
      
      test_task.delay()
      ```

    - Check Celery logs:

      ```bash
      docker-compose logs celery
      ```

      - Expected: Shows `Test task ran!`.

11. **Check Disk Usage**:

    ```bash
    df -h
    ```

    - Expected: Sufficient free space (&gt;20% free).

### 10. Finalize and Maintain

1. **Push Changes to GitHub**:

   - Commit migration files:

     ```bash
     git add .
     git commit -m "Add migration files and configs"
     git push origin main
     ```

2. **Backup Database**:

   ```bash
   docker-compose exec db pg_dump -U yourprojectname_user yourprojectname > backup.sql
   ```

3. **Automate Certificate Renewal**:

   - Test renewal:

     ```bash
     sudo certbot renew --dry-run
     ```

   - Certbot automatically renews certificates (cron job included).

4. **Monitor Logs**:

   - Periodically check:

     ```bash
     docker-compose logs --tail=10
     ```

## Troubleshooting Common Issues

1. **Containers Not Running**:

   - **Check**: `docker ps -a` to see exited containers.

   - **Fix**: View logs:

     ```bash
     docker-compose logs <service>
     ```

   - Rebuild and restart:

     ```bash
     docker-compose up -d --build
     ```

2. **Database Connection Errors**:

   - **Check**: Ensure `.env` matches `settings.py` (`DB_NAME`, `DB_USER`, `DB_PASSWORD`).

   - **Fix**: Reset database:

     ```bash
     docker-compose down -v
     docker-compose up -d
     docker-compose exec web python manage.py migrate
     ```

3. **Nginx Errors**:

   - **Check**: Test config:

     ```bash
     docker-compose exec nginx nginx -t
     ```

   - **Fix**: Verify certificate paths:

     ```bash
     ls /etc/letsencrypt/live/api.yourdomainname/
     ```

   - Restart Nginx:

     ```bash
     docker-compose restart nginx
     ```

4. **HTTPS/SSL Issues**:

   - **Check**: Test SSL:

     ```bash
     curl -v https://api.yourdomainname
     ```

   - **Fix**: Renew certificate:

     ```bash
     sudo certbot renew --dry-run
     ```

   - Re-run Certbot if needed:

     ```bash
     sudo certbot --nginx -d api.yourdomainname
     ```

5. **API 404 Errors**:

   - **Check**: Ensure `urls.py` includes DRF routes:

     ```python
     from django.urls import path, include
     from rest_framework.routers import DefaultRouter
     
     router = DefaultRouter()
     # Register your viewsets
     urlpatterns = [
         path("api/", include(router.urls)),
     ]
     ```

   - **Fix**: Restart web service:

     ```bash
     docker-compose restart web
     ```

6. **DNS Issues**:

   - **Check**: Verify DNS:

     ```bash
     dig api.yourdomainname
     ```

   - **Fix**: Wait for propagation or correct A record in your registrar.

7. **Disk Space Issues**:

   - **Check**: `df -h`

   - **Fix**: Clean up unused Docker objects:

     ```bash
     docker system prune -a --volumes
     ```

   - Resize EBS volume in AWS Console if needed.

## Additional Notes

- **Frontend Integration**: If your frontend is at `yourdomainname`, add CORS support:

  - Install `django-cors-headers`:

    ```bash
    echo "django-cors-headers" >> requirements.txt
    docker-compose up -d --build
    ```

  - Update `settings.py`:

    ```python
    INSTALLED_APPS = [
        ...,
        "corsheaders",
    ]
    
    MIDDLEWARE = [
        ...,
        "corsheaders.middleware.CorsMiddleware",
        ...,
    ]
    
    CORS_ALLOWED_ORIGINS = ["https://yourdomainname"]
    ```

- **Scaling**: For higher traffic, upgrade to a larger instance (e.g., `t2.medium`) or use AWS Elastic Beanstalk.

- **Monitoring**: Set up AWS CloudWatch for logs and alerts:

  - Navigate to CloudWatch &gt; Log Groups &gt; Create a group for Docker logs.

- **Security**:

  - Restrict SSH access to your IP in the security group.

  - Regularly update Docker images:

    ```bash
    docker-compose pull
    docker-compose up -d
    ```

## Conclusion

This guide provides a complete roadmap to host a DRF project on AWS EC2 with Docker and HTTPS. By following these steps, you can deploy a secure, scalable backend at `https://api.yourdomainname`, integrated with a frontend if needed. The verification steps ensure all components (Django, PostgreSQL, Redis, Celery, Nginx) are operational, and the troubleshooting section helps resolve common issues. Save backups of your database and certificates regularly, and monitor logs to maintain a healthy deployment.
