# Vprofile 
 Vprofile is Multi-tier web application stack (Basically a Website) written in java.Consist of following Services. 
  Nginx => Web Service
  Tomcat => Application Server
  RabbitMQ => Broker/Queuing Agent
  Memcache => DB Caching
  MySQL => SQL Database    

## 📋 Prerequisites
 Tool are Required to be installed on host machine:
- Vagrant (VM Automation tool) 
- Oracle Virtual Box (hypervisors) with different boxes
- IDE Visual Studio Code
- Git bash (CLI(Command line Interface) in Windows which let us use Unix/Linux like commands)
---

## ⚙️ Vagrant Configuration
  Vagrant Configuration Mention in Multi VM Vagrant File

---

## 🚀 Setup Instructions

### 1️⃣ Start the Virtual Machine
```bash
vagrant plugin install vagrant-hostmanager ##Installed hostplugin
vagrant up ## Followin command will up all 5 VM
vagrant status ## status of all VM
```

### 2️⃣ Access the VMs:
```bash
vagrant ssh db01
cat /etc/hosts ##Verify the host
sudo dnf update -y ##update OS with latest patches

sudo dnf install epel-release -y
sudo dnf install git mariadb-server -y ##Install Mariadb Package

sudo systemctl start mariadb
sudo systemctl enable mariadb

sudo mysql_secure_installation ##DO secure Installation of MariaDB

mysql -u root -padmin123 ##set dbname and user
mysql> create database accounts;
mysql> grant all privileges on accounts.* TO 'admin'@'localhost' identified by
'admin123';
mysql> grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
mysql> FLUSH PRIVILEGES;
mysql> exit;

cd /tmp/  ##ownload Source code and Initialize the databse
git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project

mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
mysql -u root -padmin123 accounts
systemctl restart mariadb

vagrant ssh mc01  ##Login to Memcache
cat /etc/hosts ##Verify the host
sudo dnf update -y ##update OS with latest patches

sudo dnf install epel-release -y
sudo dnf install memcached -y

sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached

sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached

vagrant ssh rmq01 ##Login to RebitMQ
cat /etc/hosts ##Verify the host
sudo dnf update -y ##update OS with latest patches

sudo dnf install wget -y
dnf -y install centos-release-rabbitmq-38
dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
systemctl enable --now rabbitmq-server
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*" sudo systemctl restart rabbitmq-server

vagrant ssh app01 ##Login to Tomcat
cat /etc/hosts ##Verify the host
sudo dnf update -y ##update OS with latest patches
dnf install epel-release -y
dnf -y install java-17-openjdk java-17-openjdk-devel
dnf install git wget -y
cd /tmp/
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar
tar xzvf apache-tomcat-10.1.26.tar.gz
useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/
cat >/etc/systemd/system/tomcat.service <<EOF
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
WantedBy=multi-user.
EOF
systemctl daemon-reload

systemctl start tomcat
systemctl enable tomcat
cd /tmp/
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
cp -r apache-maven-3.9.9 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"
git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project
 vim src/main/resources/application.properties ##update the file with Backend Details
/usr/local/maven3.9/bin/mvn install
systemctl stop tomcat
rm -rf /usr/local/tomcat/webapps/ROOT*
cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
systemctl start tomcat
chown tomcat.tomcat /usr/local/tomcat/webapps -R
systemctl restart tomcat



vagrant ssh web01
sudo -i
cat /etc/hosts ##Verify the host
sudo dnf update -y ##update OS with latest patches
apt install nginx -y
cat > /etc/nginx/sites-available/vproapp <<EOF
upstream vproapp {
server app01:8080;
}
server {
listen 80;
location / {
proxy_pass http://vproapp;
}
}
EOF
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx

```

---
## 🌐 Access the Websites
- **VprofileApplication:** [http://192.168.56.11]
- **WorkFlow:**
The user request goes to the Nginx server(used as a load balancer). It forwards the request to the Tomcat server where our application is hosted. Then it forwards that request to RabbitMQ(message broker) to Memcached for caching data. It is connected to the MySQL server where the user information is stored. If the same request comes next time it accesses the data cached in Memcached.
---
## 🛑 Stopping and Removing the VM
Stop the VM:
```bash
vagrant halt ##Poweroff all the vagrant
```
Destroy the VM:
```bash
vagrant destroy ##Remove all the VM
```
