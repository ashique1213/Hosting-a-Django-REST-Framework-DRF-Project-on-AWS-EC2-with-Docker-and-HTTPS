Documentation: Hosting a Django REST Framework (DRF) Project on AWS EC2 with Docker and HTTPS
This guide explains how to deploy a Django REST Framework (DRF) project on an AWS EC2 instance using Docker, PostgreSQL, Redis, Celery, Celery Beat, and Nginx, secured with HTTPS via Let’s Encrypt. It covers setting up the environment, generating migrations, configuring Nginx, enabling HTTPS, and verifying the deployment. The guide also includes troubleshooting steps for common issues, making it accessible to anyone with basic technical knowledge.
Prerequisites
Before starting, ensure you have:

AWS Account: Access to the AWS Management Console.
Domain Name: A registered domain (e.g., yourdomainname) for HTTPS setup.
GitHub Repository: Your DRF project hosted on GitHub, including a Dockerfile, requirements.txt, and Django project files.
SSH Client: Tools like PuTTY (Windows) or Terminal (Linux/macOS) for SSH access.
Basic Knowledge: Familiarity with Linux commands, Docker, and Django.

Project Overview

Application: A DRF backend with Django’s AbstractUser for authentication.
Components:
Web: Django app served via Daphne (ASGI).
Database: PostgreSQL for persistent data.
Redis: For Celery task queue and caching.
Celery: For background tasks.
Celery Beat: For scheduled tasks.
Nginx: Reverse proxy for HTTP/HTTPS and static file serving.


Domain:
Frontend (if applicable): yourdomainname.
Backend: api.yourdomainname.


EC2 Instance: Ubuntu with public IP (e.g., 13.127.106.180).
HTTPS: Secured with Let’s Encrypt SSL certificates.

Step-by-Step Setup Guide
1. Launch an AWS EC2 Instance

Log in to AWS Console:

Go to AWS Management Console and navigate to EC2.


Create an EC2 Instance:

Click Launch Instance.
Name: yourprojectname-server (or your choice).
OS: Choose Ubuntu Server 22.04 LTS (free tier eligible).
Instance Type: Select t2.micro (free tier) or t2.small for better performance.
Key Pair: Create or use an existing key pair (e.g., yourprojectname-key.pem) for SSH access. Download and store securely.
Network Settings:
Allow SSH (port 22) from My IP (or 0.0.0.0/0 for broader access, less secure).
Allow HTTP (port 80) and HTTPS (port 443) from 0.0.0.0/0.


Storage: Allocate at least 20GB for Docker images, volumes, and logs.
Click Launch Instance.


Get Public IP:

Once running, note the instance’s public IP (e.g., 13.127.106.180).


SSH into the Instance:

Open a terminal (or PuTTY):
ssh -i yourprojectname-key.pem ubuntu@13.127ॵ.106.180


If permission errors occur, run:
chmod 400 yourprojectname-key.pem





2. Prepare the EC2 Environment

Update the System:

Update packages and resolve any restart requirements:
sudo apt update
sudo apt upgrade -y
sudo reboot


Reconnect after reboot:
ssh -i yourprojectname-key.pem ubuntu@13.127.106.180




Install Docker and Docker Compose:

Install Docker:
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu


Install Docker Compose:
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.6/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version


Log out and back in to apply the Docker group change:
exit
ssh -i yourprojectname-key.pem ubuntu@13.127.106.180




Verify Disk Space:

Ensure sufficient space:
df -h


If low, resize the EBS volume in the AWS Console (EC2 > Volumes > Modify Volume).




3. Clone and Configure the Project

Clone the Repository:

Clone your GitHub repo:
git clone <your-github-repo-url> yourprojectname
cd yourprojectname


Replace <your-github-repo-url> with your repository URL (e.g., https://github.com/username/yourprojectname.git).



Verify Files:

Ensure the directory contains:
Dockerfile
docker-compose.yml
requirements.txt
yourprojectname/ (Django project directory)
Apps with models (e.g., users/models.py for CustomUser).




Create Environment File:

Create a .env file for secrets:
nano .env


Add:
DB_NAME=yourprojectname
DB_USER=yourprojectname_user
DB_PASSWORD=your_secure_password
REDIS_URL=redis://redis:6379/0
SECRET_KEY=your_django_secret_key
DEBUG=False
ALLOWED_HOSTS=13.127.106.180,api.yourdomainname


Generate a secure SECRET_KEY:
python -c "import secrets; print(secrets.token_urlsafe(50))"


Replace your_secure_password with a strong password.



Configure Django Settings:

Edit yourprojectname/settings.py to match the environment:
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


Ensure AUTH_USER_MODEL points to your CustomUser model (e.g., users.CustomUser).

Add rest_framework and your apps to INSTALLED_APPS.




4. Generate and Apply Migrations
If your repository lacks migration files, generate them for the database schema.

Build Docker Images:
docker-compose build


Generate Migrations:

Create migration files for all apps, including CustomUser:
docker-compose run --rm web python manage.py makemigrations


This creates migration files in app directories (e.g., users/migrations/).



Apply Migrations:
docker-compose run --rm web python manage.py migrate


Create a Superuser (optional, for admin access):
docker-compose run --rm web python manage.py createsuperuser


Follow prompts to set username, email, and password.


Verify Migrations:

Check for migration files:
ls users/migrations/


Expected: Files like 0001_initial.py.




5. Configure Nginx
Nginx serves as a reverse proxy and handles static files and HTTPS.

Create Nginx Configuration:

Create a directory and config file:
mkdir nginx
nano nginx/nginx.conf


Add the provided configuration:
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


This redirects HTTP to HTTPS, serves static files, and supports WebSocket connections (if used).



Update docker-compose.yml:

Use the provided docker-compose.yml (below) to include the Nginx service.


Collect Static Files:

Modify the web service command in docker-compose.yml to collect static files:
command: >
  sh -c "python manage.py collectstatic --noinput &&
         daphne -b 0.0.0.0 -p 8000 yourprojectname.asgi:application"


Run:
docker-compose up -d
docker-compose exec web python manage.py collectstatic --noinput





6. Set Up HTTPS with Let’s Encrypt
Secure the backend at api.yourdomainname with SSL.

Configure DNS:

Log in to your domain registrar (e.g., GoDaddy, Namecheap).

Add an A record:

Name: api
Value: 13.127.106.180 (your EC2 public IP)
TTL: Default or 300 seconds


Verify DNS propagation:
ping api.yourdomainname


Expected: Resolves to 13.127.106.180.




Install Certbot:

Install Certbot and the Nginx plugin on the EC2 instance:
sudo apt install -y certbot python3-certbot-nginx




Obtain SSL Certificate:

Run Certbot to get certificates for api.yourdomainname:
sudo certbot --nginx -d api.yourdomainname


Follow prompts:

Agree to terms.
Choose to redirect HTTP to HTTPS (recommended).


Certificates are stored in /etc/letsencrypt/live/api.yourdomainname/.



Verify Nginx Configuration:

Ensure nginx.conf references the correct certificate paths (already set in the provided config).

Test Nginx config:
docker-compose exec nginx nginx -t


Expected: syntax is ok.




Restart Services:
docker-compose down
docker-compose up -d



7. Update EC2 Security Group
Ensure the EC2 instance allows necessary traffic.

Go to AWS Console:
Navigate to EC2 > Security Groups > Select your instance’s security group.


Edit Inbound Rules:
Add or verify:
SSH: Port 22, Source: Your IP (or 0.0.0.0/0 for broader access).
HTTP: Port 80, Source: 0.0.0.0/0.
HTTPS: Port 443, Source: 0.0.0.0/0.


Remove (if unused):
Port 5432 (PostgreSQL).
Port 6379 (Redis).
Port 8000 (handled by Nginx).




Save Changes.

8. Start the Application

Run Docker Compose:
docker-compose up -d


Verify Containers:
docker ps


Expected: Six running containers:
yourprojectname_web_1
yourprojectname_db_1
yourprojectname_redis_1
yourprojectname_celery_1
yourprojectname_celery-beat_1
yourprojectname_nginx_1





9. Verify the Deployment
Check that all components are working correctly.

Check Container Status:
docker ps


Ensure all six containers are Up.


View Logs:
docker-compose logs web
docker-compose logs db
docker-compose logs redis
docker-compose logs celery
docker-compose logs celery-beat
docker-compose logs nginx


Look for:
web: Daphne listening on 0.0.0.0:8000.
db: database system is ready to accept connections.
redis: Ready to accept connections.
celery: celery@<hostname> ready.
celery-beat: Scheduler activity.
nginx: No errors; HTTP/HTTPS requests logged.




Test Database:
docker-compose exec db psql -U yourprojectname_user -d yourprojectname


Run:
\dt


Expected: Lists tables like auth_user, users_customuser.




Test Redis:
docker-compose exec redis redis-cli ping


Expected: PONG.


Test HTTP:
curl http://13.127.106.180


Expected: Redirects to HTTPS or returns a Django response.


Test HTTPS:
curl https://api.yourdomainname


Expected: Returns a Django response (e.g., DRF API root).
In a browser, visit https://api.yourdomainname:
Expected: Loads DRF browsable API or app root with a valid SSL certificate (padlock icon).




Test Admin Interface:

Visit https://api.yourdomainname/admin/ in a browser.
Log in with superuser credentials.
Expected: Admin dashboard loads, showing CustomUser and other models.


Test API Endpoints:

If you have an endpoint (e.g., /api/users/):
curl https://api.yourdomainname/api/users/


Expected: Returns JSON (e.g., [] or user data).




Test Static Files:
curl https://api.yourdomainname/static/admin/css/base.css


Expected: Returns CSS content.


Test Celery:

Open a Django shell:
docker-compose exec web python manage.py shell


Run:
from celery import shared_task
@shared_task
def test_task():
    print("Test task ran!")
    return "Done"

test_task.delay()


Check Celery logs:
docker-compose logs celery


Expected: Shows Test task ran!.




Check Disk Usage:
df -h


Expected: Sufficient free space (>20% free).



10. Finalize and Maintain

Push Changes to GitHub:

Commit migration files:
git add .
git commit -m "Add migration files and configs"
git push origin main




Backup Database:
docker-compose exec db pg_dump -U yourprojectname_user yourprojectname > backup.sql


Automate Certificate Renewal:

Test renewal:
sudo certbot renew --dry-run


Certbot automatically renews certificates (cron job included).



Monitor Logs:

Periodically check:
docker-compose logs --tail=10





Configuration Files
Below are the final, working configuration files used in this deployment.
docker-compose.yml
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

Dockerfile
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

nginx/nginx.conf
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

Troubleshooting Common Issues

Containers Not Running:

Check: docker ps -a to see exited containers.

Fix: View logs:
docker-compose logs <service>


Rebuild and restart:
docker-compose up -d --build




Database Connection Errors:

Check: Ensure .env matches settings.py (DB_NAME, DB_USER, DB_PASSWORD).

Fix: Reset database:
docker-compose down -v
docker-compose up -d
docker-compose exec web python manage.py migrate




Nginx Errors:

Check: Test config:
docker-compose exec nginx nginx -t


Fix: Verify certificate paths:
ls /etc/letsencrypt/live/api.yourdomainname/


Restart Nginx:
docker-compose restart nginx




HTTPS/SSL Issues:

Check: Test SSL:
curl -v https://api.yourdomainname


Fix: Renew certificate:
sudo certbot renew --dry-run


Re-run Certbot if needed:
sudo certbot --nginx -d api.yourdomainname




API 404 Errors:

Check: Ensure urls.py includes DRF routes:
from django.urls import path, include
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
# Register your viewsets
urlpatterns = [
    path("api/", include(router.urls)),
]


Fix: Restart web service:
docker-compose restart web




DNS Issues:

Check: Verify DNS:
dig api.yourdomainname


Fix: Wait for propagation or correct A record in your registrar.



Disk Space Issues:

Check: df -h

Fix: Clean up unused Docker objects:
docker system prune -a --volumes


Resize EBS volume in AWS Console if needed.




Additional Notes

Frontend Integration: If your frontend is at yourdomainname, add CORS support:

Install django-cors-headers:
echo "django-cors-headers" >> requirements.txt
docker-compose up -d --build


Update settings.py:
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




Scaling: For higher traffic, upgrade to a larger instance (e.g., t2.medium) or use AWS Elastic Beanstalk.

Monitoring: Set up AWS CloudWatch for logs and alerts:

Navigate to CloudWatch > Log Groups > Create a group for Docker logs.


Security:

Restrict SSH access to your IP in the security group.

Regularly update Docker images:
docker-compose pull
docker-compose up -d



