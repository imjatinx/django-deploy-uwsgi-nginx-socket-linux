# Django Deployment Guide (Nginx + uWSGI + Sockets)

This guide walks through the steps to deploy a Django project using Nginx and uWSGI with socket configuration, allowing the site to be accessible locally.

## Prerequisites

Make sure you have the following installed:
- Python 3
- pip
- Virtual Environment (`venv`)
- Nginx
- uWSGI

You can install them using the following command:

```bash
sudo apt install python3-pip python3-dev python3-venv nginx uwsgi
```

## Setup and Installation

### 1. Create and Activate a Virtual Environment

```bash
python3 -m venv ~/env/myenv
source ~/env/myenv/bin/activate
```

### 2. Install Django and uWSGI

```bash
pip install django uwsgi
```

### 3. Create a New Django Project

```bash
django-admin startproject iblogs
cd iblogs/
```

### 4. Configure Static Files

Open the `settings.py` file and add the following:

```python
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```

Generate static files:

```bash
python manage.py collectstatic
```

### 5. Create a Media Directory

```bash
mkdir media
```

### 6. Test the Django Server

To ensure everything is set up correctly, run the Django server:

```bash
python manage.py runserver 0.0.0.0:8000
```

### 7. Test Using uWSGI

```bash
uwsgi --http :8000 --module iblogs.wsgi
```

### 8. Create uWSGI Config File

Create the configuration file to run uWSGI on a socket:

```bash
nano iblogs_uwsgi.ini
```

Paste the following configuration:

```ini
[uwsgi]
chdir = /home/imjatinx/iblogs/
module = iblogs.wsgi

home = /home/imjatinx/env/myenv
master = true
processes = 5
socket = /home/imjatinx/iblogs/iblogs.sock
chmod-socket = 666
vacuum = true
daemonize = /home/imjatinx/iblogs/uwsgi-emperor.log
```

### 9. Test the uWSGI Configuration

```bash
uwsgi --ini iblogs_uwsgi.ini
```

### 10. Configure uWSGI Emperor as a Service

This configuration will auto-restart on reboot and manage multiple applications.

#### Create Vassals Directory

```bash
sudo mkdir -p /etc/uwsgi/vassals
```

#### Create a Symlink

```bash
sudo ln -s /home/imjatinx/iblogs/iblogs_uwsgi.ini /etc/uwsgi/vassals/
```

#### Create Emperor Service File

```bash
sudo nano /etc/systemd/system/emperor.uwsgi.service
```

Paste the following content:

```ini
[Unit]
Description=uWSGI Emperor for Django iblogs microdomains
After=network.target

[Service]
Restart=always
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

### 11. Reload Services

```bash
sudo systemctl daemon-reload
sudo systemctl restart emperor.uwsgi.service
sudo systemctl status emperor.uwsgi.service
```

## Nginx Configuration

### 1. Create an Nginx Configuration File

```bash
sudo nano /etc/nginx/sites-available/iblogs.conf
```

Paste the following configuration:

```nginx
server {
    listen 80;
    server_name 192.168.29.12;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/imjatinx/iblogs/iblogs.sock;
    }

    location /static/ {
        alias /home/imjatinx/iblogs/static/;
    }

    location /media/ {
        alias /home/imjatinx/iblogs/media/;
    }

    error_log /home/imjatinx/iblogs/nginx-error.log;
    access_log /home/imjatinx/iblogs/nginx-access.log;
}
```

### 2. Test Nginx Syntax

```bash
sudo nginx -t
```

### 3. Enable the Nginx Site

```bash
sudo ln -s /etc/nginx/sites-available/iblogs.conf /etc/nginx/sites-enabled/
```

### 4. Restart Nginx

```bash
sudo systemctl restart nginx.service
```

## Conclusion

Once all the steps are complete, navigate to Server IP in your browser to see your Django project running!
