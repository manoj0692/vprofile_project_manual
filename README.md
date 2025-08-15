# Vprofile - Multi-tier Java Web Application
Vprofile is a multi-tier web application stack written in Java, consisting of interconnected microservices that simulate real-world e-commerce infrastructure.
  
## 📋 Prerequisites
 Tool are Required to be installed on host machine:
- Vagrant (VM Automation tool) 
- Oracle Virtual Box (hypervisors) with different boxes
- IDE (Visual Studio Code)
- Git bash (CLI(Command line Interface) in Windows which let us use Unix/Linux like commands)
---

 ## ⚙️ Infrastructure Components(Vagrant Configuration)
| Service    | VM   | IP Address     | Host OS         | Memory | Ports     |
|------------|------|----------------|-----------------|--------|-----------|
| Nginx      | web01| 192.168.56.11 | Ubuntu 22.04    | 800MB  | 80        |
| Tomcat     | app01| 192.168.56.12 | CentOS Stream 9 | 800MB  | 8080      |
| RabbitMQ   | rmq01| 192.168.56.13 | CentOS Stream 9 | 600MB  | 5672,15672|
| Memcached  | mc01 | 192.168.56.14 | CentOS Stream 9 | 600MB  | 11211     |
| MySQL      | db01 | 192.168.56.15 | CentOS Stream 9 | 600MB  | 3306      |


---

## 🚀 Setup Instructions
Setup should be done in below mentioned order
- MySQL (Database SVC)
- Memcache (DB Caching SVC)
- RabbitMQ (Broker/Queue SVC)
- Tomcat (Application SVC)
- Nginx (Web SVC)


### 1️⃣ Start the Virtual Machine
```bash
# Installed Vagrant Plugin hostmanager 
vagrant plugin install vagrant-hostmanager
# Start All VM 
vagrant up
# Status of VMs
vagrant status ## status of all VM
```
### Database Setup (db01)
```bash
vagrant ssh db01
sudo dnf update -y
sudo dnf install epel-release git mariadb-server -y
sudo systemctl enable --now mariadb

# Secure installation (set password: admin123)
sudo mysql_secure_installation

# Configure database
mysql -u root -padmin123 <<EOF
CREATE DATABASE accounts;
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'localhost' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';
FLUSH PRIVILEGES;
EXIT;
EOF

# Initialize database
cd /tmp
git clone -b local https://github.com/hkhcoder/vprofile-project.git
mysql -u root -padmin123 accounts < vprofile-project/src/main/resources/db_backup.sql
sudo systemctl restart mariadb
```
### Memcached Application Setup VM (mc01)
```bash
vagrant ssh mc01
sudo dnf update -y
sudo dnf install epel-release memcached -y
sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl enable --now memcached
```
### Rabbitmq Application Setup (rmq01)
```bash
vagrant ssh rmq01
sudo dnf update -y
sudo dnf install wget -y
sudo dnf -y install centos-release-rabbitmq-38
sudo dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
sudo systemctl enable --now rabbitmq-server

# Configure access
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
sudo systemctl restart rabbitmq-server
```
 ### Tomcat Application Setup (app01)
```bash
vagrant ssh app01
sudo dnf update -y
sudo dnf install epel-release java-17-openjdk-devel git wget -y

# Install Tomcat
cd /tmp
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
tar xzvf apache-tomcat-10.1.26.tar.gz
sudo useradd -m -d /usr/local/tomcat -s /bin/false tomcat
sudo cp -r apache-tomcat-10.1.26/* /usr/local/tomcat/

# Create systemd service
sudo tee /etc/systemd/system/tomcat.service > /dev/null <<'EOF'
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
User=tomcat
Group=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_PID=/usr/local/tomcat/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINA_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Build and deploy application
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
sudo unzip apache-maven-3.9.9-bin.zip -d /usr/local/
export PATH="/usr/local/apache-maven-3.9.9/bin:$PATH"

git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project

# Update application properties
cat <<'PROP' | sudo tee src/main/resources/application.properties
#JDBC Configuration
spring.datasource.url=jdbc:mysql://db01:3306/accounts?useSSL=false
spring.datasource.username=admin
spring.datasource.password=admin123

#Memcached Configuration
memcached.active.host=mc01
memcached.active.port=11211

#RabbitMQ Configuration
rabbitmq.address=rmq01
rabbitmq.port=5672
rabbitmq.username=test
rabbitmq.password=test
PROP

mvn clean install
sudo systemctl stop tomcat
sudo rm -rf /usr/local/tomcat/webapps/ROOT*
sudo cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
sudo chown -R tomcat:tomcat /usr/local/tomcat/webapps
sudo systemctl start tomcat
```


 ### Nginx Application Setup (web01)
```bash
vagrant ssh web01
sudo apt update && sudo apt install nginx -y

# Configure reverse proxy
sudo tee /etc/nginx/sites-available/vproapp > /dev/null <<'EOF'
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
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

---
## 🌐 Access the Websites
- **VprofileApplication:** Access through Web01  [http://192.168.56.11]
- **WorkFlow:**
The user request goes to the Nginx server web01 (used as a load balancer). It forwards the request to the Tomcat server where our application is hosted. after that itforwards that request to RabbitMQ(message broker) to Memcached for caching data. It is connected to the MySQL server where the user information is stored. If the same request comes next time it accesses the data cached in Memcached.
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
