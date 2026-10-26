# Fullstack Deployment Guide: React Frontend on AWS EC2 with Nginx & Cloudflare

<div align="center">

[![GitHub Release](https://img.shields.io/github/v/release/codewithmuh/frontend-react?color=blue&logo=github)](https://github.com/codewithmuh/frontend-react/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Stars](https://img.shields.io/github/stars/codewithmuh/frontend-react?style=flat&color=gold)](https://github.com/codewithmuh/frontend-react/stargazers)
[![GitHub Forks](https://img.shields.io/github/forks/codewithmuh/frontend-react?style=flat&color=teal)](https://github.com/codewithmuh/frontend-react/network/members)
[![React](https://img.shields.io/badge/Frontend-React_SPA-61DAFB?logo=react)](https://react.dev/)
[![AWS EC2](https://img.shields.io/badge/Compute-AWS_EC2_Ubuntu-FF9900?logo=amazon-aws)](https://aws.amazon.com/ec2/)
[![Nginx](https://img.shields.io/badge/Reverse_Proxy-Nginx-009639?logo=nginx)](https://nginx.org/)
[![Cloudflare](https://img.shields.io/badge/DNS_%26_SSL-Cloudflare-F38020?logo=cloudflare)](https://www.cloudflare.com/)
[![Backend: Django](https://img.shields.io/badge/Paired_Backend-Django_DRF-092E20?logo=django)](https://github.com/codewithmuh/backend-django)
[![YouTube: @codewithmuh](https://img.shields.io/badge/YouTube-codewithmuh-FF0000?logo=youtube&logoColor=white)](https://www.youtube.com/@codewithmuh)

**Complete deployment architecture and production guide for hosting a React frontend alongside a Django REST Framework backend on a single AWS EC2 server behind Nginx, Gunicorn, and Cloudflare.**

[Architecture](#system-architecture) • [EC2 Setup](#1-aws-ec2-instance-setup) • [Elastic IP & DNS](#2-attaching-elastic-ip--cloudflare-dns) • [Frontend Build](#4-frontend-react-build--deployment) • [Nginx Configuration](#5-nginx-unified-reverse-proxy-configuration) • [Paired Backend](#paired-backend-repository)

</div>

---

![Deployment Architecture Banner](https://github.com/user-attachments/assets/b80ba865-2dc1-4cce-bce9-add85715edc7)

## Overview

This repository hosts the **React Single Page Application (SPA)** designed to run collaboratively with the [`backend-django`](https://github.com/codewithmuh/backend-django) REST API on a single AWS EC2 instance. This unified single-host architecture minimizes infrastructure costs while maximizing performance by serving optimized static frontend bundles directly through Nginx and reverse-proxying API traffic to Gunicorn.

---

## System Architecture

```mermaid
flowchart TD
    subgraph Clients["Users & Browsers"]
        Browser["User Web Browser"]
    end

    subgraph Edge["Edge Network & CDN"]
        Cloudflare["Cloudflare (Universal SSL + DNS Proxy)"]
    end

    subgraph AWSCloud["AWS Infrastructure (EC2 Ubuntu 22.04 LTS)"]
        EIP["AWS Elastic IP (Static Public IPv4)"]
        
        subgraph ServerHost["Single Host Nginx Server"]
            Nginx["Nginx Web Server / Reverse Proxy (Port 80 / 443)"]
            StaticFiles["React Production Build (/var/www/frontend/build)"]
            
            subgraph BackendService["Django Gunicorn Daemon"]
                Socket["Unix Domain Socket / Localhost:8000"]
                Django["Django REST Framework API"]
            end
        end
    end

    Browser -->|HTTPS Request| Cloudflare
    Cloudflare -->|Proxied Traffic| EIP
    EIP --> Nginx
    Nginx -->|Route / (Static HTML/JS)| StaticFiles
    Nginx -->|Route /api/* & /admin/*| Socket
    Socket --> Django
```

---

## Step-by-Step Production Deployment Guide

### 1. AWS EC2 Instance Setup
1. Log into AWS Console and navigate to **EC2 Dashboard**.
2. Launch a new instance with **Ubuntu Server 22.04 LTS** (e.g., `t2.micro` or `t3.small`).
3. Configure Security Group Rules:
   - **SSH (22):** Your personal IP only.
   - **HTTP (80):** Anywhere (`0.0.0.0/0`) or Cloudflare IP ranges.
   - **HTTPS (443):** Anywhere (`0.0.0.0/0`) or Cloudflare IP ranges.

### 2. Attaching Elastic IP & Cloudflare DNS
1. Navigate to **Elastic IPs** in EC2 and allocate a new IPv4 address.
2. Associate the Elastic IP with your newly created instance.
3. In your **Cloudflare Dashboard**, add an `A` record pointing your domain (e.g., `example.com`) to the Elastic IP address with the proxy status set to **Proxied (Orange Cloud)**.
4. Set SSL/TLS encryption mode to **Full** or **Full (Strict)**.

### 3. Server Package Installation
SSH into your server and install required dependencies:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx git curl build-essential

# Install Node.js (LTS version)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### 4. Frontend React Build & Deployment
Clone the frontend repository and compile production static assets:
```bash
# Clone to web root directory
sudo mkdir -p /var/www/frontend
sudo chown -R $USER:$USER /var/www/frontend
git clone https://github.com/codewithmuh/frontend-react.git /var/www/frontend

# Install dependencies and build
cd /var/www/frontend
npm install
npm run build
```

### 5. Nginx Unified Reverse Proxy Configuration
Create an Nginx server block to serve React static files and route `/api/` traffic to the Django backend:

```bash
sudo nano /etc/nginx/sites-available/fullstack-app
```

Paste the following configuration:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    # React Static Assets
    root /var/www/frontend/build;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Django Backend API Proxy
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Django Admin Panel
    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Django Static / Media Files
    location /static/ {
        alias /var/www/backend/staticfiles/;
    }

    location /media/ {
        alias /var/www/backend/media/;
    }
}
```

Enable the configuration and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/fullstack-app /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## Paired Backend Repository

For complete instructions on setting up Python, Django, Gunicorn, and Systemd background services on the same host, refer to the paired repository:
👉 [**codewithmuh/backend-django**](https://github.com/codewithmuh/backend-django)

---

## Community & Video Walkthrough

- 📺 **YouTube Channel:** [CodeWithMuh](https://www.youtube.com/@codewithmuh)
- 🌐 **Connect on LinkedIn:** [Muhammad Rashid](https://www.linkedin.com/in/codewithmuh)

---

## License

This project is licensed under the [MIT License](LICENSE).
