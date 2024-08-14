# Self-Hosted Dify: Perfect AI App Builder

Welcome to the guide on setting up and using Dify, a powerful open-source platform for building AI applications. This document will walk you through the process of self-hosting Dify on a Google Cloud VM using Docker and Docker Compose, and how to trigger workflows using APIs.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Installation Steps](#installation-steps)
  - [Install Required Packages](#install-required-packages)
  - [Add Docker's Official GPG Key](#add-dockers-official-gpg-key)
  - [Set Up the Stable Repository](#set-up-the-stable-repository)
  - [Update Package Index](#update-package-index)
  - [Install Docker](#install-docker)
  - [Install Nginx and Set Up SSL](#install-nginx-and-set-up-ssl)
  - [Install Dify and Reconfigure Nginx](#install-dify-and-reconfigure-nginx)
- [Upgrading Dify](#upgrading-dify)
- [Accessing Dify](#accessing-dify)
- [Triggering Workflows via API](#triggering-workflows-via-api)

## Introduction

Dify is an open-source platform designed to make building large language model (LLM) applications easy and accessible. With its intuitive interface and powerful features, Dify allows developers and non-technical users alike to create AI applications without writing complex code.

## Prerequisites

Before starting, ensure you have:

- A virtual machine (VM) instance running Debian or Ubuntu
- Access to your VM via SSH
- A domain name pointing to your server's public IP

## Installation Steps

### Install Required Packages

Update your package list and install the necessary packages:

```bash
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
```
### Add Docker's Official GPG Key
Add Docker's official GPG key to your system:
```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```

### Set Up the Stable Repository
Add the Docker repository to your package sources. Replace debian with ubuntu if you're using Ubuntu:
```bash
echo "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
For Ubuntu, replace debian with ubuntu.

### Update your package index and install Docker:
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### Install Nginx and Set Up SSL

First, install Nginx:
```bash
sudo apt-get install nginx
```
After installation, visit your server's public IP address in a browser to see the Nginx welcome page.

Next, install Certbot for SSL certificate management:
```bash
sudo apt install certbot python3-certbot-nginx
```

Ensure you have a domain name with a DNS record pointing to your server's public IP. Then, run Certbot to obtain and install the SSL certificate:
```bash
sudo certbot --nginx
```
Provide your email, agree to the terms, and specify your domain(s). After completion, your site should be accessible via HTTPS with a valid SSL certificate.

### Install Dify and Reconfigure Nginx

Clone the Dify repository:
```bash
git clone https://github.com/langgenius/dify.git
```
Navigate to the Dify directory and set up the environment:
```bash
cd dify/docker
cp .env.example .env
```

Modify the .env file to change the Nginx ports:
```bash
# Original ports
# EXPOSE_NGINX_PORT=80
# EXPOSE_NGINX_SSL_PORT=443

# New ports
EXPOSE_NGINX_PORT=8080
EXPOSE_NGINX_SSL_PORT=8443
```

Reconfigure Nginx to forward traffic to Dify. First, back up the existing configuration:
```bash
cd /etc/nginx/sites-available
sudo cp default default.bkp
```

Open the default file and comment out the following sections:
```bash
root /var/www/html;
index index.html index.htm index.nginx-debian.html;
location / {
     try_files $uri $uri/ =404;
}
```

Why do we need to comment these?
`root /var/www/html;`:
This directive sets the root directory from which Nginx will serve files. When active, it tells Nginx to look for files in /var/www/html to serve in response to requests.

`index index.html index.htm index.nginx-debian.html;`:
This specifies the default file to serve if a directory is requested. For example, if someone visits http://your_domain.com/, Nginx will look for an index.html or similar file in the root directory specified above.

`location / { try_files $uri $uri/ =404; }`:
This block instructs Nginx to attempt to serve a file corresponding to the requested URI. If no such file is found, it tries to serve a directory or returns a 404 error if neither is available.

Add a new location block to forward traffic to Dify:
```bash
location / {
    proxy_pass http://localhost:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_redirect off;
    proxy_buffering off;
    proxy_request_buffering off;
    client_max_body_size 0;
}
```

Check the Nginx configuration for errors:
```bash
sudo nginx -t
```

If successful, restart Nginx:
```bash
sudo systemctl restart nginx
```

If you visit your site and see a "bad gateway" error, this is expected until Dify is running.

### Get Dify Running

From the dify/docker directory, run:
```bash
cd ~/dify/docker
docker compose up -d
```

### Check Deployment Results
Verify that all containers are running successfully:
```bash
docker compose ps
```

This includes 3 core services: api / worker / web, and 6 dependent components: weaviate / db / redis / nginx / ssrf_proxy / sandbox .

```bash
NAME                  IMAGE                              COMMAND                   SERVICE      CREATED              STATUS                        PORTS
docker-api-1          langgenius/dify-api:0.6.13         "/bin/bash /entrypoi…"   api          About a minute ago   Up About a minute             5001/tcp
docker-db-1           postgres:15-alpine                 "docker-entrypoint.s…"   db           About a minute ago   Up About a minute (healthy)   5432/tcp
docker-nginx-1        nginx:latest                       "sh -c 'cp /docker-e…"   nginx        About a minute ago   Up About a minute             0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp
docker-redis-1        redis:6-alpine                     "docker-entrypoint.s…"   redis        About a minute ago   Up About a minute (healthy)   6379/tcp
docker-sandbox-1      langgenius/dify-sandbox:0.2.1      "/main"                   sandbox      About a minute ago   Up About a minute             
docker-ssrf_proxy-1   ubuntu/squid:latest                "sh -c 'cp /docker-e…"   ssrf_proxy   About a minute ago   Up About a minute             3128/tcp
docker-weaviate-1     semitechnologies/weaviate:1.19.0   "/bin/weaviate --hos…"   weaviate     About a minute ago   Up About a minute             
docker-web-1          langgenius/dify-web:0.6.13         "/bin/sh ./entrypoin…"   web          About a minute ago   Up About a minute             3000/tcp
docker-worker-1       langgenius/dify-api:0.6.13         "/bin/bash /entrypoi…"   worker       About a minute ago   Up About a minute             5001/tcp
```

### Accessing Dify
Access Dify by navigating to http://<your-domain> in your web browser.

## Triggering Workflows via API
Once Dify is set up, you can trigger workflows using the API. Here's an example curl command to trigger a workflow:
```bash
curl -X POST 'https://api.dify.ai/v1/workflows/run' \
--header 'Authorization: Bearer <YOUR_AUTH_TOKEN>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "inputs": {},
    "response_mode": "streaming",
    "user": "abc-123"
}'
```
## Upgrading Dify
Enter the docker directory of the dify source code and execute the following commands:
```bash
cd dify/docker
docker compose down
git pull origin main
docker compose pull
docker compose up -d
```

## Conclusion
Congratulations! You've successfully set up Dify on your Google Cloud VM and learned how to trigger workflows using the API. Enjoy building powerful AI applications with Dify!