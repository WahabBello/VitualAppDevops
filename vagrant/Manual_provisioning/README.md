# 📘 Manual Installation Guide - VitualApp Project

## 📋 Table of Contents
- [Overview](#-overview)
- [Services Architecture](#️-services-architecture)
- [Starting VMs](#-starting-vms)
- [Services Configuration](#️-services-configuration)
  - [1. MySQL](#1-mysql---database)
  - [2. Memcache](#2-memcache---database-caching)
  - [3. RabbitMQ](#3-rabbitmq---message-broker)
  - [4. Tomcat](#4-tomcat---application-server)
  - [5. Nginx](#5-nginx---web-server)
- [Verification](#-verification)

---

## 🎯 Overview

This guide walks you through the manual configuration of the VitualApp infrastructure, service by service. This approach is ideal for understanding each system component in detail.

**⚠️ Important:** Services must be configured in the order indicated below to ensure proper application functionality.

---

## 🏗️ Services Architecture

| Service | Role | Hostname | Port |
|---------|------|----------|------|
| **MySQL** | SQL Database | db01 | 3306 |
| **Memcache** | Database Caching | mc01 | 11211 |
| **RabbitMQ** | Broker/Queue Agent | rmq01 | 5672 |
| **Tomcat** | Application Server | app01 | 8080 |
| **Nginx** | Web Server / Reverse Proxy | web01 | 80 |

---

## 🚀 Starting VMs

Navigate to the project directory:

```bash
cd vagrant/Manual_provisioning
```

Start all virtual machines:

```bash
vagrant up
```

**📝 Notes:**
- Startup may take several minutes depending on your configuration
- If the process is interrupted, simply rerun `vagrant up`
- All VM hostnames and /etc/hosts file entries will be automatically updated

---

## ⚙️ Services Configuration

### 1. MySQL - Database

#### Connect to the VM
```bash
vagrant ssh db01
sudo -i
```

#### Verify hosts entries
```bash
cat /etc/hosts
```

#### Update the system
```bash
dnf update -y
dnf install epel-release -y
```

#### Install MariaDB
```bash
dnf install git mariadb-server -y
systemctl start mariadb
systemctl enable mariadb
```

#### Secure MySQL
```bash
mysql_secure_installation
```

**Recommended configuration:**
- Set root password: `Y` → Use `admin123`
- Remove anonymous users: `Y`
- Disallow root login remotely: `n`
- Remove test database: `Y`
- Reload privilege tables: `Y`

#### Create the database
```bash
mysql -u root -padmin123
```

```sql
CREATE DATABASE accounts;
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'localhost' IDENTIFIED BY 'admin123';
FLUSH PRIVILEGES;
EXIT;
```

#### Initialize data
```bash
cd /tmp/
git clone -b local https://github.com/WahabBello/VitualAppDevops.git
cd VitualAppDevops
mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
```

**Verification:**
```bash
mysql -u root -padmin123 accounts
```
```sql
SHOW TABLES;
EXIT;
```

#### Configure firewall
```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
systemctl restart mariadb
```

---

### 2. Memcache - Database Caching

#### Connect to the VM
```bash
vagrant ssh mc01
sudo -i
```

#### Verify hosts entries
```bash
cat /etc/hosts
```

#### Install Memcached
```bash
dnf update -y
dnf install memcached -y
systemctl start memcached
systemctl enable memcached
```

#### Configure to listen on all interfaces
```bash
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
systemctl restart memcached
```

#### Configure firewall
```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port=11211/tcp
firewall-cmd --add-port=11111/udp
firewall-cmd --runtime-to-permanent
```

#### Start Memcached
```bash
memcached -p 11211 -U 11111 -u memcached -d
```

**Verification:**
```bash
systemctl status memcached
```

---

### 3. RabbitMQ - Message Broker

#### Connect to the VM
```bash
vagrant ssh rmq01
sudo -i
```

#### Verify hosts entries
```bash
cat /etc/hosts
```

#### Update the system
```bash
dnf update -y
```

#### Install RabbitMQ
```bash
dnf install wget -y
dnf -y install centos-release-rabbitmq-38
dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
systemctl enable --now rabbitmq-server
```

#### Configure user
```bash
sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
systemctl restart rabbitmq-server
```

#### Configure firewall
```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port=5672/tcp
firewall-cmd --runtime-to-permanent
```

**Verification:**
```bash
systemctl status rabbitmq-server
```

---

### 4. Tomcat - Application Server

#### Connect to the VM
```bash
vagrant ssh app01
sudo -i
```

#### Verify hosts entries
```bash
cat /etc/hosts
```

#### Install dependencies
```bash
dnf update -y
dnf -y install java-17-openjdk java-17-openjdk-devel
dnf install git wget -y
```

#### Download and install Tomcat
```bash
cd /tmp/
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
tar xzvf apache-tomcat-10.1.26.tar.gz
```

#### Create Tomcat user
```bash
useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
```

#### Configure Tomcat
```bash
cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/
chown -R tomcat.tomcat /usr/local/tomcat
```

#### Create systemd service
```bash
vi /etc/systemd/system/tomcat.service
```

Add the following content:
```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
User=tomcat
Group=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINE_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

#### Start Tomcat
```bash
systemctl daemon-reload
systemctl start tomcat
systemctl enable tomcat
```

#### Configure firewall
```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --reload
```

#### Install Maven
```bash
cd /tmp/
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
cp -r apache-maven-3.9.9 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"
```

#### Download source code
```bash
git clone -b local https://github.com/WahabBello/VitualAppDevops.git
cd VitualAppDevops
```

#### Update configuration
```bash
vim src/main/resources/application.properties
```

**Update with backend server details:**
- Database: `db01:3306`
- Memcache: `mc01:11211`
- RabbitMQ: `rmq01:5672`

#### Build the application
```bash
/usr/local/maven3.9/bin/mvn install
```

#### Deploy
```bash
systemctl stop tomcat
rm -rf /usr/local/tomcat/webapps/ROOT*
cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
chown tomcat.tomcat /usr/local/tomcat/webapps -R
systemctl start tomcat
systemctl restart tomcat
```

---

### 5. Nginx - Web Server

#### Connect to the VM
```bash
vagrant ssh web01
sudo -i
```

#### Verify hosts entries
```bash
cat /etc/hosts
```

#### Update the system
```bash
apt update
apt upgrade -y
```

#### Install Nginx
```bash
apt install nginx -y
```

#### Configure reverse proxy
```bash
vi /etc/nginx/sites-available/vproapp
```

Add the following content:
```nginx
upstream vproapp {
    server app01:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://vproapp;
    }
}
```

#### Enable configuration
```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

---

## ✅ Verification

Once all services are configured, verify the application:

**Access the application:**

Open your browser and navigate to:
```
http://192.168.56.21
```

**Check logs in case of issues:**
```bash
# Tomcat logs
tail -f /usr/local/tomcat/logs/catalina.out

# Nginx logs
tail -f /var/log/nginx/error.log
```

---

**🎉 Congratulations!** You have manually configured the entire VitualApp infrastructure.