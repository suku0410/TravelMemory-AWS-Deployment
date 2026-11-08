# TravelMemory — MERN Application Deployment on AWS

## 1\. Project Overview

TravelMemory is a full-stack MERN application deployed on AWS EC2. It uses a React frontend, a Node.js/Express backend, and MongoDB Atlas.

The deployment contains two frontend EC2 instances and two backend EC2 instances. Each tier is registered with its own Application Load Balancer (ALB).

## 2\. Technology Stack

* Frontend: React
* Backend: Node.js and Express
* Database: MongoDB Atlas
* Web server / reverse proxy: Nginx
* Process management: systemd
* Cloud platform: AWS EC2
* Load balancing: AWS Application Load Balancer

## 3\. AWS Infrastructure

|Resource|Configuration|
|-|-|
|AWS Region|`us-east-1` (US East – N. Virginia)|
|VPC|`travel\_memory\_vpc`|
|VPC CIDR|`10.0.0.0/16`|
|Subnet 01|`travel\_memory\_subnet01` – `10.0.0.0/20`|
|Subnet 02|`travel\_memory\_subnet02` – `10.0.16.0/24`|
|Frontend instances|`frontend-master`, `frontend-2`|
|Backend instances|`backend-master`, `backend-2`|
|Frontend target group|`travelmemory-frontend-tg`|
|Backend target group|`travelmemory-backend-tg`|
|Frontend ALB|`travelmemory-frontend-alb`|
|Backend ALB|`travelmemory-backend-alb`|

### Load balancer endpoints used during deployment

* Frontend ALB: `http://travelmemory-frontend-alb-746295185.us-east-1.elb.amazonaws.com`
* Backend ALB: `http://travelmemory-backend-alb-219947082.us-east-1.elb.amazonaws.com`

These are the DNS names recorded during deployment. They will change if the load balancers are deleted and recreated.

## 4\. Repository Structure

```text
TravelMemory-AWS-Deployment/
├── backend/                  # Node.js / Express API
├── frontend/                 # React application
├── docs/
│   ├── architecture/         # AWS deployment architecture diagram
│   └── SOP/                  # Step-by-step deployment SOP
├── README.md
├── .gitignore
└── backend/.env.example
```

## 5\. Architecture Diagram

The AWS deployment architecture diagram is maintained under:

`docs/architecture/TravelMemory\_Deployment\_Architecture.png`

The diagram represents the application request flow:

```text
Users / Browser
      |
Frontend Application Load Balancer
      |
Frontend EC2 Instances (2)
      |
Backend Application Load Balancer
      |
Backend EC2 Instances (2)
      |
MongoDB Atlas
```

The architecture includes the AWS VPC, subnet ranges, frontend and backend target groups, both ALBs, four EC2 instances, and MongoDB Atlas. Cloudflare is not included because a custom domain was not configured for this deployment.

## 6\. Deployment SOP

The deployment Standard Operating Procedure (SOP) is maintained under:

`docs/SOP/TravelMemory\_MERN\_Deployment\_Guide.docx`

The SOP documents the deployment process in the following order:

1. MongoDB Atlas setup.
2. Prerequisites: VPC creation, subnet configuration, Internet Gateway and route table, security groups, and two Ubuntu 22.04 EC2 instances.
3. Backend master configuration: package installation, backend `.env`, systemd service, and Nginx reverse proxy.
4. Frontend master configuration: package installation, API URL configuration, React build, and Nginx.
5. Preparing the base AMI images.
6. Launching the second frontend and backend instances from AMIs.
7. Creating frontend and backend target groups.
8. Creating frontend and backend Application Load Balancers.
9. Pointing the frontend to the backend ALB and validating application connectivity.

## 7\. Backend Configuration

### 7.1 Environment variables

Create a file named `.env` inside `backend/` on each backend EC2 instance:

```env
MONGO\_URI=mongodb+srv://<username>:<password>@<cluster-host>/travelmemory?retryWrites=true\&w=majority
PORT=3000
CORS\_ORIGIN=http://travelmemory-frontend-alb-746295185.us-east-1.elb.amazonaws.com
```

* `MONGO\_URI`: MongoDB Atlas connection string.
* `PORT`: Port on which the Node.js backend listens.
* `CORS\_ORIGIN`: Frontend ALB origin allowed by the backend CORS configuration.

**Security:** Never commit the real `.env` file or MongoDB credentials. Commit only an `.env.example` file containing placeholders.

### 7.2 Install backend dependencies

Run on both backend EC2 instances:

```bash
sudo apt update
sudo apt install -y git nginx
# Install a Node.js version compatible with the project dependencies.
# The deployed environment used Node.js v22.23.4.

git clone <YOUR\_GITHUB\_REPOSITORY\_URL>
cd TravelMemory-AWS-Deployment/backend
npm install
```

### 7.3 systemd service

Create `/etc/systemd/system/travelmemory-backend.service`:

```ini
\[Unit]
Description=TravelMemory Backend
After=network.target

\[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/TravelMemory-AWS-Deployment/backend
EnvironmentFile=/home/ubuntu/TravelMemory-AWS-Deployment/backend/.env
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=5

\[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now travelmemory-backend
sudo systemctl status travelmemory-backend
```

Restart after changing `.env`:

```bash
sudo systemctl restart travelmemory-backend
```

### 7.4 Nginx reverse proxy

Nginx listens on HTTP port 80 and forwards requests to the Node.js application on `localhost:3000`. Configure the server block in `/etc/nginx/sites-available/default`:

```nginx
server {
    listen 80;
    server\_name \_;

    location / {
        proxy\_pass http://localhost:3000;
        proxy\_set\_header Host $host;
        proxy\_set\_header X-Real-IP $remote\_addr;
        proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
    }
}
```

Validate and restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```

Test:

```bash
curl -i http://localhost:3000/hello
```

The deployment test returned HTTP `200 OK` and `Hello World!`.

## 8\. Frontend Configuration

### 8.1 Backend API URL

The React frontend sends API requests to the backend ALB.

In `frontend/src/url.js`:

```javascript
export const baseUrl = "http://travelmemory-backend-alb-219947082.us-east-1.elb.amazonaws.com";
```

If the backend ALB is recreated, update this value to the new backend ALB DNS name and rebuild the frontend.

### 8.2 Install dependencies and build

Run on both frontend EC2 instances:

```bash
sudo apt update
sudo apt install -y git nginx

git clone <YOUR\_GITHUB\_REPOSITORY\_URL>
cd TravelMemory-AWS-Deployment/frontend
npm install
npm run build
```

Publish the build files to Nginx:

```bash
sudo rm -rf /var/www/html/\*
sudo cp -r build/\* /var/www/html/
sudo systemctl restart nginx
sudo systemctl status nginx
```

## 9\. Load Balancer and Target Group Configuration

### Frontend

* Target group: `travelmemory-frontend-tg`
* Protocol and port: HTTP `80`
* Health check path: `/`
* Registered targets: `frontend-master`, `frontend-2`
* Load balancer: `travelmemory-frontend-alb`
* Listener: HTTP `80`, forwarding to the frontend target group

### Backend

* Target group: `travelmemory-backend-tg`
* Protocol and port: HTTP `80`
* Health check path: `/hello`
* Registered targets: `backend-master`, `backend-2`
* Load balancer: `travelmemory-backend-alb`
* Listener: HTTP `80`, forwarding to the backend target group

Both target groups showed two healthy registered targets during deployment validation.

## 10\. CORS and Application Communication

The browser loads the React application from the frontend ALB. API calls from the browser go to the backend ALB. The backend `.env` must allow the frontend ALB origin:

```env
CORS\_ORIGIN=http://travelmemory-frontend-alb-746295185.us-east-1.elb.amazonaws.com
```

If the frontend endpoint or protocol changes, update `CORS\_ORIGIN` on both backend instances and restart the backend service.

## 11\. Deployment Validation

The following checks were completed during deployment:

* Backend systemd service was active.
* Nginx was active on frontend and backend instances.
* Backend logs confirmed connection to MongoDB Atlas.
* `curl http://localhost:3000/hello` returned HTTP `200 OK`.
* Backend target group showed two healthy targets.
* Frontend target group showed two healthy targets.
* Frontend loaded through the frontend ALB DNS name.
* Trip records were displayed and a new experience was saved successfully.

## 12\. Deployment Notes

This repository contains the TravelMemory frontend and backend application source code, AWS configuration notes, architecture diagram, and deployment SOP. Before publishing, check that no credentials, `.env` files, private keys, or other sensitive information are included.

