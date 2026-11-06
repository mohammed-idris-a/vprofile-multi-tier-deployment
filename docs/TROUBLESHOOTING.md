# VProfile Deployment – Troubleshooting and Resolutions

This document records the issues encountered while provisioning and deploying the VProfile multi-tier application, the troubleshooting steps performed, and the solutions that worked.

The purpose is to document practical implementation experience rather than only the standard deployment procedure.

---

## 1. Vagrant Base Box Provisioning Issues

### Symptoms

During the initial VM provisioning process, the following errors occurred.

**Issue 1 – Box checksum mismatch**

```text
Downloading successful, but checksum doesn't match.
```

**Issue 2 – BaseOS repository metadata retrieval failure**

```text
Error: Failed to download metadata for repo 'baseos':
Cannot download repomd.xml:
Cannot download repodata/repomd.xml:
All mirrors were tried
```

### Investigation and Resolution

The base box reference for the RabbitMQ VM was updated in the `Vagrantfile`.

Previous configuration was replaced with:

```ruby
rmq01.vm.box = "generic/rocky9"
```

### Result

After updating the box reference, VM provisioning completed successfully.

**Note:** The two errors were observed during the initial provisioning process. The box-reference update was the change that resulted in successful provisioning in this environment; the errors are not necessarily proof of a single underlying cause.

---

## 2. Nginx 502 Bad Gateway – Tomcat Backend Unreachable

### Symptoms

After configuring Nginx as the web server and reverse proxy, accessing the application resulted in:

```text
502 Bad Gateway
```

Nginx was configured to forward requests to the Tomcat application server at:

```text
http://app01:8080
```

### Investigation

The first step was to check whether the web server could resolve the application server hostname.

From `web01`:

```bash
ping -c 2 app01
```

Next, a direct connection to the Tomcat backend was attempted:

```bash
curl -v http://app01:8080
```

The connection failed:

```text
Trying 192.168.56.12:8080...
* connect to 192.168.56.12 port 8080 failed: No route to host
* Failed to connect to app01 port 8080 after 3 ms: No route to host
* Closing connection 0
curl: (7) Failed to connect to app01 port 8080
```

The hostname resolved to `192.168.56.12`, but the connection to TCP port 8080 failed.

### Root Cause

The firewall on `app01` was blocking incoming connections to Tomcat's port 8080.

### Resolution

Logged in to the application server:

```bash
vagrant ssh app01
```

Checked the firewall configuration:

```bash
sudo firewall-cmd --list-all
```

Allowed Tomcat traffic on TCP port 8080 permanently:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### Verification

Checked whether the Tomcat service was running:

```bash
sudo systemctl status tomcat
```

Checked whether Tomcat was listening on port 8080:

```bash
sudo ss -tulnp | grep 8080
```

The firewall change resolved the backend connectivity problem.

---

## 3. Application Login Failure – Database Connectivity

### Symptoms

After configuring the application and database, the application login did not work with the user credentials available in the database.

The application was unable to communicate with the database service as expected.

### Investigation and Resolution

The database server uses MariaDB, which listens on TCP port 3306.

The firewall on the database VM was preventing the required database connection.

Logged in to the database VM and allowed TCP port 3306:

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

### Result

After allowing database traffic through the firewall, the database connectivity issue was resolved and application login worked.

---

## 4. RabbitMQ and Memcached Firewall Workaround

### Issue

During the project setup, firewall restrictions on the RabbitMQ and Memcached VMs were another potential source of service connectivity problems.

### Workaround Applied

To avoid further firewall-related interruptions during this lab deployment, the firewalls on the following VMs were disabled:

- `rmq01` – RabbitMQ
- `mc01` – Memcached

This was a workaround used in the isolated project environment, not a recommended production configuration.

### Recommended Firewall Configuration

A more secure approach is to keep the firewall enabled and allow only the service ports required by the application.

**RabbitMQ – TCP 5672**

```bash
sudo firewall-cmd --permanent --add-port=5672/tcp
sudo firewall-cmd --reload
```

**Memcached – TCP 11211**

```bash
sudo firewall-cmd --permanent --add-port=11211/tcp
```

**Memcached – UDP 11111, if UDP is enabled in the service configuration**

```bash
sudo firewall-cmd --permanent --add-port=11111/udp
sudo firewall-cmd --reload
```

For a production environment, access should also be restricted to the specific application hosts that require these services.

---

## 5. Key Troubleshooting Takeaways

The issues encountered during deployment highlighted several practical lessons:

1. **VM provisioning:** Base box compatibility and package repository availability can affect provisioning.
2. **Backend connectivity:** A working hostname resolution does not guarantee that the required application port is reachable.
3. **Reverse proxy troubleshooting:** When Nginx returns a 502 error, test connectivity to the upstream application server directly.
4. **Service verification:** Check service status and listening ports before assuming an application configuration problem.
5. **Database connectivity:** Verify that the database service is reachable from the application server and that the required firewall rules are configured.
6. **Firewall management:** Opening only the required ports is preferable to disabling the firewall.

These troubleshooting steps were performed during the VProfile deployment using the project's Vagrant-based multi-VM environment.
