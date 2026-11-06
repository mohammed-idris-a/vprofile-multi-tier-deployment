# VProfile – Deployment Guide

This runbook documents the local VProfile multi-tier deployment using Vagrant, VirtualBox, and five Linux virtual machines. It follows the course setup sequence and records the environment-specific changes and workarounds used during this implementation.

> **Scope:** This is a learning/lab deployment, not a production-ready reference architecture. Commands and service versions may need adjustment when using different Vagrant boxes or Linux releases.

## 1. Architecture and deployment order

The application is deployed across five VMs:

| VM | Service | Main port | Purpose |
|---|---|---:|---|
| `db01` | MariaDB | TCP 3306 | Stores application data in the `accounts` database |
| `mc01` | Memcached | TCP 11211 | Caching service |
| `rmq01` | RabbitMQ | TCP 5672 | Demonstration message-broker service; not functionally integrated in this course version |
| `app01` | Apache Tomcat | TCP 8080 | Runs the Java web application |
| `web01` | Nginx | TCP 80 | Receives browser requests and reverse-proxies them to Tomcat |

Configure and validate the services in this order:

1. MariaDB (`db01`)
2. Memcached (`mc01`)
3. RabbitMQ (`rmq01`)
4. Tomcat and application deployment (`app01`)
5. Nginx (`web01`)

See [ARCHITECTURE.md](ARCHITECTURE.md) for the request flow, VM roles, and network connections.

## 2. Prerequisites on the host

Install the following on the host machine:

- Oracle VirtualBox
- Vagrant
- Git for Windows / Git Bash, or an equivalent terminal
- A text editor

Install the Vagrant Host Manager plugin:

```bash
vagrant plugin install vagrant-hostmanager
```

Check the installations:

```bash
vagrant --version
vagrant plugin list
```

The Host Manager plugin updates host entries so that the VMs can resolve names such as `db01`, `app01`, and `web01` without manually maintaining each VM's `/etc/hosts` file.

## 3. Clone the repository and provision the VMs

Clone this repository on the host:

```bash
git clone https://github.com/mohammed-idris-a/vprofile-multi-tier-deployment.git
cd vprofile-multi-tier-deployment
```

Change to the directory containing the `Vagrantfile`. In the course layout this is:

```bash
cd vagrant/Manual_provisioning
```

If the directory has changed, locate the `Vagrantfile` and run the following commands from its directory.

Start the VMs:

```bash
vagrant up
```

If provisioning is interrupted, inspect the VM state and run `vagrant up` again for the VMs that still need provisioning:

```bash
vagrant status
vagrant up
```

Confirm that the expected VMs are running:

```bash
vagrant status
```

The Host Manager plugin should populate hostname mappings. From a VM, verify name resolution:

```bash
getent hosts db01 mc01 rmq01 app01 web01
```

### Environment-specific base-box adjustment

During initial provisioning, the original base-box reference for `rmq01` produced a checksum mismatch and a BaseOS repository metadata retrieval error. The `rmq01` entry in the `Vagrantfile` was changed to:

```ruby
rmq01.vm.box = "generic/rocky9"
```

Provisioning completed successfully after this change in the lab environment. This adjustment applies to `rmq01`; it does not imply that the base boxes for all five VMs were changed.

## 4. Configure MariaDB on `db01`

Connect to the database VM from the directory containing the `Vagrantfile`:

```bash
vagrant ssh db01
```

Update the operating system and install MariaDB:

```bash
sudo dnf update -y
sudo dnf install -y epel-release git mariadb-server
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

Run the MariaDB hardening script and set a lab password when prompted:

```bash
sudo mysql_secure_installation
```

Create the application database and application account. Replace the placeholder with a lab password; do not paste real credentials into this document or commit them to Git:

```bash
sudo mysql -u root -p
```

Run the following SQL statements in the MariaDB prompt:

```sql
CREATE DATABASE accounts;
CREATE USER 'admin'@'%' IDENTIFIED BY '<DB_PASSWORD>';
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%';
FLUSH PRIVILEGES;
EXIT;
```

The `%` host pattern is a broad lab convenience. For a hardened environment, restrict the account to the application server's source IP or an explicitly defined subnet.

Clone this repository on `db01` so the database dump is available:

```bash
cd /tmp
git clone --branch main https://github.com/mohammed-idris-a/vprofile-multi-tier-deployment.git
```

Import the supplied database dump:

```bash
sudo mysql -u root -p accounts < /tmp/vprofile-multi-tier-deployment/src/main/resources/db_backup.sql
```

Verify that the tables were imported:

```bash
sudo mysql -u root -p -e "USE accounts; SHOW TABLES;"
```

Allow the application server to reach MariaDB on TCP port 3306:

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

The deployment troubleshooting record explains the database connectivity/login problem that was resolved by allowing this port.

## 5. Configure Memcached on `mc01`

Connect to the Memcached VM:

```bash
vagrant ssh mc01
```

Install and start Memcached:

```bash
sudo dnf update -y
sudo dnf install -y epel-release memcached
```

The course configuration changes the bind address so the service can accept connections from other VMs:

```bash
sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl enable --now memcached
sudo systemctl restart memcached
sudo systemctl status memcached
```

Verify the listening socket:

```bash
sudo ss -lntup | grep 11211
```

The application configuration points to `mc01` on TCP port 11211. During this lab, the firewall on `mc01` was disabled as a workaround to avoid further connectivity interruptions. That is a lab-only choice, not a recommended security configuration.

A safer alternative is to leave the firewall enabled and allow only the required Memcached traffic from the application VM. For example, to permit TCP 11211:

```bash
sudo firewall-cmd --permanent --add-port=11211/tcp
sudo firewall-cmd --reload
```

Do not expose Memcached to untrusted networks. Restrict access to the application network/host when applying firewall rules.

## 6. Configure RabbitMQ on `rmq01`

Connect to the RabbitMQ VM:

```bash
vagrant ssh rmq01
```

Install the packages and start RabbitMQ using the repository/package procedure supplied with the course environment:

```bash
sudo dnf update -y
sudo dnf install -y epel-release wget
sudo dnf -y install centos-release-rabbitmq-38
sudo dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
sudo systemctl enable --now rabbitmq-server
sudo systemctl status rabbitmq-server
```

Create a RabbitMQ lab account if required by the local application configuration. Replace the placeholder password and do not commit it:

```bash
sudo rabbitmqctl add_user test '<RABBITMQ_PASSWORD>'
sudo rabbitmqctl set_user_tags test administrator
sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
```

Verify the service and listener:

```bash
sudo systemctl status rabbitmq-server
sudo ss -lntp | grep 5672
```

RabbitMQ is included as a demonstration component in this version of the course project. The course walkthrough explicitly describes it as not functionally integrated into the application's normal workflow.

During this lab, the firewall on `rmq01` was disabled as a workaround. This is not a production recommendation. The preferred approach is to keep the firewall enabled and allow TCP port 5672 only from the application VM/network if RabbitMQ is required:

```bash
sudo firewall-cmd --permanent --add-port=5672/tcp
sudo firewall-cmd --reload
```

## 7. Configure Tomcat on `app01`

Connect to the application VM:

```bash
vagrant ssh app01
```

Install Java 17 and the required utilities:

```bash
sudo dnf update -y
sudo dnf install -y epel-release java-17-openjdk java-17-openjdk-devel git wget unzip
java -version
```

Download and extract the Tomcat version used by the course setup:

```bash
cd /tmp
wget -O apache-tomcat-10.1.26.tar.gz https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
tar -xzf apache-tomcat-10.1.26.tar.gz
```

Create the Tomcat service account and install the Tomcat files:

```bash
sudo useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
sudo mkdir -p /usr/local/tomcat
sudo cp -a /tmp/apache-tomcat-10.1.26/. /usr/local/tomcat/
sudo chown -R tomcat:tomcat /usr/local/tomcat
```

Determine the Java installation path and create `/etc/systemd/system/tomcat.service`:

```bash
readlink -f "$(command -v java)"
```

Use the matching Java 17 home directory for `JAVA_HOME` in this systemd unit (adjust that one path if the host reports a different location):

```ini
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=simple
User=tomcat
Group=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINA_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Save the file, then reload systemd and start Tomcat:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat
sudo systemctl status tomcat
```

Allow inbound TCP port 8080 so that Nginx on `web01` can reach Tomcat:

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Verify that Tomcat is listening:

```bash
sudo ss -lntp | grep 8080
```

## 8. Build and deploy the Java application on `app01`

Install Maven 3.9.9 as used in the setup guide:

```bash
cd /tmp
wget -O apache-maven-3.9.9-bin.zip https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
sudo cp -a apache-maven-3.9.9 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"
export PATH="/usr/local/maven3.9/bin:$PATH"
mvn -version
```

Clone the project source on `app01`:

```bash
cd /tmp
git clone --branch main https://github.com/mohammed-idris-a/vprofile-multi-tier-deployment.git
cd vprofile-multi-tier-deployment
```

Review `src/main/resources/application.properties` and configure the service endpoints for this lab:

| Setting | Expected endpoint |
|---|---|
| JDBC database | `db01:3306`, database `accounts` |
| Memcached | `mc01:11211` |
| RabbitMQ | `rmq01:5672` (demonstration component) |

Set the database and service credentials to match the accounts created on the respective VMs. Keep real credentials in a local, untracked configuration or a secrets mechanism; do not commit passwords to a public repository.

Build the WAR from the directory containing `pom.xml`:

```bash
mvn install
```

Confirm that the build produced the expected artifact:

```bash
ls -lh target/vprofile-v2.war
```

Deploy the WAR as the Tomcat root application:

```bash
sudo systemctl stop tomcat
sudo rm -rf /usr/local/tomcat/webapps/ROOT*
sudo cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
sudo chown -R tomcat:tomcat /usr/local/tomcat/webapps
sudo systemctl start tomcat
sudo systemctl status tomcat
```

Check the Tomcat logs if the application does not deploy:

```bash
sudo journalctl -u tomcat -n 100 --no-pager
```

## 9. Configure Nginx on `web01`

Connect to the web VM:

```bash
vagrant ssh web01
```

The course environment uses Ubuntu commands on `web01`:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y nginx
```

Create `/etc/nginx/sites-available/vproapp`:

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

Disable the default site and enable the VProfile site:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
```

Validate the configuration before reloading Nginx:

```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx
sudo systemctl status nginx
```

## 10. End-to-end verification

Perform checks from the relevant VMs rather than assuming that a running service is reachable from every tier.

From `web01`, check name resolution and the Tomcat backend:

```bash
getent hosts app01
curl -v http://app01:8080
```

From `app01`, verify the database and cache endpoints are resolvable and the relevant ports are reachable. For example:

```bash
getent hosts db01 mc01
```

From `web01`, test Nginx locally:

```bash
curl -I http://localhost
```

Finally, open the web VM's reachable IP address in a browser:

```text
http://<WEB01_IP>
```

Verify that the VProfile application page loads and that the login flow works with the database-backed account used in the lab. Do not include working usernames/passwords in public documentation.

For the specific failures encountered during this deployment—including the Nginx 502 response and the database login/connectivity problem—see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## 11. Stop or destroy the lab

Run these commands from the directory containing the `Vagrantfile`:

Stop the VMs but retain their disks:

```bash
vagrant halt
```

Destroy the VMs and their local state:

```bash
vagrant destroy -f
```

`vagrant destroy -f` is destructive. It removes the VMs created by this Vagrant environment; ensure that anything you need from the VMs has been saved first.

## 12. Implementation notes

- The `rmq01` base-box reference was changed to `generic/rocky9` after initial provisioning errors; successful provisioning was verified afterward.
- TCP 8080 was opened on `app01` after direct testing from `web01` showed that Nginx could not reach Tomcat.
- TCP 3306 was opened on `db01` after the application could not complete the database-backed login flow.
- The firewalls on `mc01` and `rmq01` were disabled as lab workarounds. This is not a secure production pattern; prefer narrowly scoped firewall rules.
- The commands above are a runbook for this lab. Review package repository availability and service versions before reusing them on a different Linux release.
