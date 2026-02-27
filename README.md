# Cloud Computing Lab — EC2 Flask Application with RDS & DynamoDB

**Institution:** Fr. Conceicao Rodrigues College of Engineering
**Department:** Computer Engineering | T.E. VI — Cloud Computing Lab

## Overview

This repository contains the deployment of a Flask web application on AWS EC2, demonstrating connections to both relational (Amazon RDS) and NoSQL (Amazon DynamoDB) databases.

* **Lab 4 / Lab 5 Assignment 1:** Microblog (Flask) on EC2 + Amazon RDS MySQL
* **Lab 5 Assignment 2:** DynamoBlog (Flask) on EC2 + Amazon DynamoDB

### Application Details

* **EC2 Public IP:** `44.192.56.170`
* **Microblog URL:** `http://44.192.56.170/`
* **DynamoBlog URL:** `http://44.192.56.170:8001/`
* **Region:** `us-east-1` (N. Virginia)
* **Instance:** `t3.micro` | Ubuntu 22.04 LTS

---

## Repository Structure

```text
/
├── microblog/          # Lab 4 + Assignment 1 — Flask Microblog app (RDS MySQL)
│   ├── app/
│   ├── migrations/
│   ├── microblog.py
│   └── requirements.txt
├── dynamoblog/         # Assignment 2 — Flask DynamoBlog app (DynamoDB)
│   └── app.py
└── README.md

```

---

## Lab 4 — Microblog on EC2 (Flask + Gunicorn + Nginx)

### What Was Deployed

* Flask Microblog app from `miguelgrinberg/microblog`
* Gunicorn WSGI server running locally on `127.0.0.1:8000`
* Nginx reverse proxy routing traffic on port `80`
* `systemd` service for auto-start on instance reboot

### EC2 Setup Details

* **AMI:** Ubuntu 22.04 LTS (`ami-0030e4319cbf4dbf2`)
* **Instance type:** `t3.micro` (Free Tier)
* **Storage:** 20 GB EBS
* **Security Group Inbound Rules:**
* Port `22` (SSH) → My IP only
* Port `80` (HTTP) → `0.0.0.0/0`



### Deployment Steps

**1. Update system and install dependencies**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip git nginx build-essential

```

**2. Clone the repository**

```bash
cd ~
git clone https://github.com/miguelgrinberg/microblog.git
cd microblog

```

**3. Create virtual environment and install dependencies**

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install gunicorn

```

**4. Initialize SQLite database (Lab 4 baseline)**

```bash
export FLASK_APP=microblog.py
export SECRET_KEY=your-secret-key
flask db upgrade

```

**5. Test locally**

```bash
flask run --host=127.0.0.1 --port=5000
curl -I http://127.0.0.1:5000

```

### Configurations

**systemd Service (`/etc/systemd/system/microblog.service`)**

```ini
[Unit]
Description=Microblog Flask Application
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/microblog
Environment="FLASK_APP=microblog.py"
Environment="SECRET_KEY=your-secret-key"
Environment="DATABASE_URL=mysql+pymysql://admin:<password>@microblog-db.cy5swyyy4z97.us-east-1.rds.amazonaws.com:3306/microblog"
ExecStart=/home/ubuntu/microblog/venv/bin/gunicorn --workers 4 --bind 127.0.0.1:8000 microblog:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

```

**Service Activation**

```bash
sudo systemctl daemon-reload
sudo systemctl enable microblog
sudo systemctl start microblog
sudo systemctl status microblog

```

**Nginx Configuration (`/etc/nginx/sites-available/microblog`)**

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/microblog_access.log;
    error_log /var/log/nginx/microblog_error.log;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

**Nginx Activation**

```bash
sudo ln -s /etc/nginx/sites-available/microblog /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx

```

---

## Lab 5 Assignment 1 — Connect Microblog to Amazon RDS MySQL

### Database Settings

| Setting | Value |
| --- | --- |
| **Engine** | MySQL 8.4.7 |
| **Instance class** | db.t4g.micro (Free Tier) |
| **Storage** | 20 GB gp2 |
| **Database name** | microblog |
| **Endpoint** | microblog-db.cy5swyyy4z97.us-east-1.rds.amazonaws.com |
| **Port** | 3306 |
| **Multi-AZ** | No |
| **Public access** | No |

### RDS Setup and Integration Steps

**1. Install MySQL client and Python driver on EC2**

```bash
sudo apt install -y mysql-client
cd ~/microblog && source venv/bin/activate
pip install pymysql cryptography

```

**2. Test connection and create database**

```bash
mysql -h microblog-db.cy5swyyy4z97.us-east-1.rds.amazonaws.com -u admin -p
# Inside MySQL prompt:
CREATE DATABASE microblog;
show databases;
exit;

```

**3. Set DATABASE_URL and apply schema**

```bash
export FLASK_APP=microblog.py
export DATABASE_URL='mysql+pymysql://admin:<password>@microblog-db.cy5swyyy4z97.us-east-1.rds.amazonaws.com:3306/microblog'
flask db upgrade

```

**4. Verify tables and update service**

```bash
mysql -h microblog-db.cy5swyyy4z97.us-east-1.rds.amazonaws.com -u admin -p microblog
show tables;
exit;

# Update systemd service with DATABASE_URL and restart
sudo systemctl daemon-reload
sudo systemctl restart microblog

```

### Security & Database Details

* **Security Group (`rds-ec2-1`):** Inbound TCP port `3306` is allowed **only** from the EC2 Security Group (`ec2-rds-1`). No `0.0.0.0/0` access.
* **Credential Management:** `DATABASE_URL` is securely stored as an environment variable in the `systemd` service file.
* **Schema (9 Tables Created):** `user`, `post`, `followers`, `message`, `notification`, `task`, `token`, `alembic_version`.

### CRUD Demonstration

| Operation | Method |
| --- | --- |
| **Create** | Register new user + create new post via web UI |
| **Read** | View home feed and user profile (data retrieved from RDS) |
| **Update** | Edit Profile → update "About Me" field |
| **Delete** | Executed `DELETE FROM post WHERE id=1` via MySQL terminal |

---

## Lab 5 Assignment 2 — DynamoBlog on EC2 with DynamoDB

### What Was Deployed

* A new Flask app (DynamoBlog) running on Gunicorn via port `8001`.
* Amazon DynamoDB table named `microblog-posts`.
* IAM Role `EC2-DynamoDB-Role` attached to the EC2 instance for credential-free AWS API access.

### DynamoDB Table Design

| Setting | Value |
| --- | --- |
| **Table name** | microblog-posts |
| **Partition key** | post_id (String) |
| **Sort key** | created_at (String) |
| **Capacity mode** | On-demand |
| **Region** | us-east-1 |

### Five DynamoDB Attribute Types Used

| Attribute | Type | Example |
| --- | --- | --- |
| `post_id` | String (S) | "550e8400-e29b-41d4..." |
| `likes` | Number (N) | 5 |
| `published` | Boolean (BOOL) | true |
| `tags` | List (L) | ["flask", "dynamodb", "aws"] |
| `metadata` | Map (M) | {"app": "dynamoblog", "version": "1.0"} |

### DynamoDB + IAM Setup Steps

**1. Create IAM Role and Attach to EC2**

* Create an IAM Role (`EC2-DynamoDB-Role`) with the `AmazonDynamoDBFullAccess` policy and AWS EC2 as the trusted entity.
* Attach this role to the EC2 instance via the AWS Console (Actions → Security → Modify IAM role).

**2. Set up DynamoBlog app on EC2**

```bash
cd ~
mkdir dynamoblog && cd dynamoblog
python3 -m venv venv
source venv/bin/activate
pip install flask boto3 gunicorn
# (Create app.py in this directory)

```

**3. Create and Start systemd service (`/etc/systemd/system/dynamoblog.service`)**

```ini
[Unit]
Description=DynamoBlog Flask DynamoDB Application
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/dynamoblog
ExecStart=/home/ubuntu/dynamoblog/venv/bin/gunicorn --workers 4 --bind 0.0.0.0:8001 app:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

```

```bash
sudo systemctl daemon-reload
sudo systemctl enable dynamoblog
sudo systemctl start dynamoblog

```

### Security & API Details

* **Security:** Accessed exclusively via the IAM Role. No hardcoded AWS access keys exist in the code (`boto3` retrieves credentials automatically from instance metadata). EC2 Security Group allows port `8001` inbound.
* **CRUD Endpoints:**

| Operation | Endpoint / Method | Description |
| --- | --- | --- |
| **Create** | POST `/post` | Creates new post via `put_item()` |
| **Read** | GET `/posts` | Fetches all posts via `table.scan()` |
| **Update** | POST `/post/update` | Increments likes via `update_item()` with `UpdateExpression` |
| **Delete** | POST `/post/delete` | Removes item via `delete_item()` using `post_id` + `created_at` |

**Sample Item JSON (Datatype Proof)**

```json
{
  "post_id": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2026-02-26T19:01:43.123456",
  "author": "Fredrick",
  "title": "My First DynamoDB Post",
  "body": "Hello from DynamoBlog on EC2",
  "likes": 5,
  "published": true,
  "tags": ["flask", "dynamodb", "aws"],
  "metadata": {
    "app": "dynamoblog",
    "version": "1.0",
    "region": "us-east-1"
  }
}

```

---

## Software Versions

| Software | Version |
| --- | --- |
| **OS** | Ubuntu 22.04 LTS |
| **Python** | 3.10.x |
| **Flask** | 3.x |
| **Gunicorn** | 21.2.0 / 25.1.0 |
| **Nginx** | 1.18.0 |
| **MySQL (RDS)** | 8.4.7 |
| **boto3** | latest |
| **pymysql** | 1.1.2 |

---

## References

* Miguel Grinberg — Flask Microblog
* Flask Mega Tutorial Part XVII — Linux Deployment
* AWS RDS Free Tier Documentation
* AWS DynamoDB Documentation
* boto3 DynamoDB Documentation

