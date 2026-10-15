# AWS EC2 Production Deployment Guide

Real-world setup for deploying a Node.js application on AWS EC2 (Ubuntu Linux) with systemd, Apache reverse proxy, cron jobs, and CloudWatch monitoring.

> Based on production deployment at Navat Technology — AI Voice Commentary Platform

---

## Stack

- **Cloud:** AWS EC2 (Ubuntu 22.04 LTS)
- **Web Server:** Apache2 (reverse proxy)
- **Process Manager:** systemd
- **Storage:** AWS S3
- **Monitoring:** AWS CloudWatch
- **Database:** PostgreSQL + Redis

---

## 1. EC2 Instance Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js (v18 LTS)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install Apache
sudo apt install -y apache2

# Install PM2 (optional, we use systemd here)
# Enable required Apache modules
sudo a2enmod proxy proxy_http rewrite headers
sudo systemctl restart apache2
```

---

## 2. systemd Service Config

Create `/etc/systemd/system/app.service`:

```ini
[Unit]
Description=Node.js Production App
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/app
ExecStart=/usr/bin/node /home/ubuntu/app/dist/index.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nodeapp
Environment=NODE_ENV=production
EnvironmentFile=/home/ubuntu/app/.env

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable app
sudo systemctl start app
sudo systemctl status app
```

---

## 3. Apache Reverse Proxy

Create `/etc/apache2/sites-available/app.conf`:

```apache
<VirtualHost *:80>
    ServerName yourdomain.com

    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/

    ErrorLog ${APACHE_LOG_DIR}/app_error.log
    CustomLog ${APACHE_LOG_DIR}/app_access.log combined
</VirtualHost>
```

```bash
sudo a2ensite app.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

---

## 4. Cron Jobs

Edit with `crontab -e`:

```cron
# Match sync — every 30 minutes
*/30 * * * * /home/ubuntu/app/scripts/match-sync.sh >> /var/log/cron/match-sync.log 2>&1

# Scorecard update — every 15 minutes
*/15 * * * * /home/ubuntu/app/scripts/scorecard.sh >> /var/log/cron/scorecard.log 2>&1

# Audio cleanup — every 10 minutes
*/10 * * * * /home/ubuntu/app/scripts/audio-cleanup.sh >> /var/log/cron/audio-cleanup.log 2>&1

# Job retry — every 5 minutes
*/5 * * * * /home/ubuntu/app/scripts/job-retry.sh >> /var/log/cron/job-retry.log 2>&1
```

```bash
# Create log directory
sudo mkdir -p /var/log/cron
sudo chown ubuntu:ubuntu /var/log/cron
```

---

## 5. AWS S3 — IAM Credentials

```bash
# Install AWS CLI
sudo apt install -y awscli

# Configure (use IAM user with S3-only permissions)
aws configure
# AWS Access Key ID: ***
# AWS Secret Access Key: ***
# Default region: ap-south-1
# Output format: json
```

IAM policy for least-privilege S3 access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

---

## 6. CloudWatch — Basic Monitoring

```bash
# Install CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

CloudWatch agent config (`/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`):

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "app/syslog",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"]
      },
      "disk": {
        "measurement": ["disk_used_percent"]
      }
    }
  }
}
```

---

## 7. Security Checklist

- [ ] IAM user with least-privilege (no AdministratorAccess)
- [ ] `.env` file never committed to Git (add to `.gitignore`)
- [ ] Security Group: only ports 22, 80, 443 open
- [ ] SSH key-pair access only (no password login)
- [ ] S3 bucket — Block Public Access enabled
- [ ] CloudWatch alarms set for CPU > 80%

---

## Architecture

Internet
│
▼
EC2 Security Group (port 80/443)
│
▼
Apache2 Reverse Proxy (:80)
│
▼
Node.js App (:3000) ← systemd (auto-restart)
│
├── PostgreSQL (RDS or local)
├── Redis (ElastiCache or local)
└── AWS S3 (audio storage)
│
└── CloudWatch (logs + metrics)


---

## Author

**Abishek S** — DevOps Engineer  
[LinkedIn](https://linkedin.com/in/abishek-iam) · [GitHub](https://github.com/abisheksug)
