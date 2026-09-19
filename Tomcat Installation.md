# Tutorial

@ ways to inatall 

1. With script 
2. Manual

# Using script 

```bash
#!/usr/bin/env bash

# ============================================================
# Apache Tomcat 10.1 - One Click Installer
# Ubuntu / Debian / AWS EC2
#
# Usage:
#   sudo bash install-tomcat.sh
#
# With custom passwords:
#   sudo bash install-tomcat.sh 'ManagerPassword' 'AdminPassword'
#
# Optional specific Tomcat version:
#   sudo TOMCAT_VERSION=10.1.60 bash install-tomcat.sh
#
# ============================================================

set -Eeuo pipefail

# ------------------------------------------------------------
# Configuration
# ------------------------------------------------------------

TOMCAT_USER="tomcat"
TOMCAT_GROUP="tomcat"

TOMCAT_HOME="/opt/tomcat"
TOMCAT_PORT="8080"

# Leave empty to automatically detect latest Tomcat 10.1.x
TOMCAT_VERSION="${TOMCAT_VERSION:-}"

MANAGER_USERNAME="manager"
ADMIN_USERNAME="admin"

MANAGER_PASSWORD="${1:-ChangeMe_Manager_123!}"
ADMIN_PASSWORD="${2:-ChangeMe_Admin_123!}"

DOWNLOAD_DIR="/tmp/tomcat-install"

# ------------------------------------------------------------
# Functions
# ------------------------------------------------------------

log() {
    echo
    echo "============================================================"
    echo "$1"
    echo "============================================================"
}

error_exit() {
    echo
    echo "ERROR: $1"
    echo
    exit 1
}

cleanup() {
    rm -rf "${DOWNLOAD_DIR}"
}

trap cleanup EXIT

trap 'echo; echo "ERROR: Installation failed at line $LINENO"; exit 1' ERR

# ------------------------------------------------------------
# Root check
# ------------------------------------------------------------

if [[ "${EUID}" -ne 0 ]]; then
    error_exit "Please run this script with sudo or as root."
fi

# ------------------------------------------------------------
# OS check
# ------------------------------------------------------------

if [[ ! -f /etc/os-release ]]; then
    error_exit "/etc/os-release not found."
fi

source /etc/os-release

echo
echo "Operating System:"
echo "${PRETTY_NAME:-Unknown}"

if [[ "${ID}" != "ubuntu" && "${ID}" != "debian" ]]; then
    echo
    echo "WARNING: This script is designed for Ubuntu/Debian."
    echo "Detected: ${ID:-unknown}"
fi

# ------------------------------------------------------------
# Install required packages
# ------------------------------------------------------------

log "Installing required packages"

export DEBIAN_FRONTEND=noninteractive

apt-get update -y

apt-get install -y \
    default-jdk \
    curl \
    wget \
    ca-certificates \
    tar \
    gzip \
    grep \
    sed \
    systemd \
    ufw \
    mawk

# ------------------------------------------------------------
# Verify required commands
# ------------------------------------------------------------

log "Checking required commands"

REQUIRED_COMMANDS=(
    java
    curl
    wget
    tar
    grep
    sed
    awk
)

for COMMAND in "${REQUIRED_COMMANDS[@]}"; do

    if ! command -v "${COMMAND}" >/dev/null 2>&1; then
        error_exit "Required command '${COMMAND}' was not found."
    fi

done

echo "All required commands are available."

# ------------------------------------------------------------
# Check Java
# ------------------------------------------------------------

log "Checking Java installation"

if ! command -v java >/dev/null 2>&1; then
    error_exit "Java installation failed."
fi

java -version

JAVA_BIN="$(readlink -f "$(command -v java)")"

JAVA_HOME="$(dirname "$(dirname "${JAVA_BIN}")")"

if [[ ! -d "${JAVA_HOME}" ]]; then
    error_exit "Could not determine JAVA_HOME."
fi

echo
echo "JAVA_HOME=${JAVA_HOME}"

# ------------------------------------------------------------
# Detect latest Tomcat 10.1
# ------------------------------------------------------------

if [[ -z "${TOMCAT_VERSION}" ]]; then

    log "Detecting latest Tomcat 10.1 version"

    TOMCAT_DOWNLOAD_PAGE="https://tomcat.apache.org/download-10.cgi"

    TOMCAT_VERSION="$(
        curl -fsSL "${TOMCAT_DOWNLOAD_PAGE}" |
        grep -oE '10\.1\.[0-9]+' |
        sort -V |
        tail -1
    )"

    if [[ -z "${TOMCAT_VERSION}" ]]; then
        error_exit "Unable to determine the latest Tomcat 10.1 version."
    fi

else

    log "Using specified Tomcat version"

fi

echo
echo "Tomcat Version: ${TOMCAT_VERSION}"

# ------------------------------------------------------------
# Validate Tomcat version
# ------------------------------------------------------------

if [[ ! "${TOMCAT_VERSION}" =~ ^10\.1\.[0-9]+$ ]]; then
    error_exit "Invalid Tomcat version: ${TOMCAT_VERSION}"
fi

# ------------------------------------------------------------
# Build download URL
# ------------------------------------------------------------

TOMCAT_TARBALL="apache-tomcat-${TOMCAT_VERSION}.tar.gz"

TOMCAT_URL="https://dlcdn.apache.org/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/${TOMCAT_TARBALL}"

ARCHIVE_URL="https://archive.apache.org/dist/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/${TOMCAT_TARBALL}"

# ------------------------------------------------------------
# Prepare download directory
# ------------------------------------------------------------

log "Preparing download directory"

rm -rf "${DOWNLOAD_DIR}"

mkdir -p "${DOWNLOAD_DIR}"

cd "${DOWNLOAD_DIR}"

# ------------------------------------------------------------
# Download Tomcat
# ------------------------------------------------------------

log "Downloading Apache Tomcat ${TOMCAT_VERSION}"

echo
echo "Primary URL:"
echo "${TOMCAT_URL}"
echo

if ! curl \
    --fail \
    --location \
    --show-error \
    --progress-bar \
    --retry 3 \
    --retry-delay 5 \
    -o "${TOMCAT_TARBALL}" \
    "${TOMCAT_URL}"; then

    echo
    echo "Primary Apache mirror failed."
    echo "Trying Apache archive..."

    rm -f "${TOMCAT_TARBALL}"

    echo
    echo "Archive URL:"
    echo "${ARCHIVE_URL}"
    echo

    curl \
        --fail \
        --location \
        --show-error \
        --progress-bar \
        --retry 3 \
        --retry-delay 5 \
        -o "${TOMCAT_TARBALL}" \
        "${ARCHIVE_URL}"

fi

# ------------------------------------------------------------
# Validate download
# ------------------------------------------------------------

if [[ ! -s "${TOMCAT_TARBALL}" ]]; then
    error_exit "Tomcat download failed."
fi

echo
echo "Downloaded:"
ls -lh "${TOMCAT_TARBALL}"

# ------------------------------------------------------------
# Validate tar archive
# ------------------------------------------------------------

log "Validating Tomcat archive"

if ! tar -tzf "${TOMCAT_TARBALL}" >/dev/null 2>&1; then
    error_exit "Downloaded Tomcat archive is invalid."
fi

echo "Tomcat archive validation successful."

# ------------------------------------------------------------
# Stop existing Tomcat
# ------------------------------------------------------------

log "Stopping existing Tomcat service"

if systemctl list-unit-files 2>/dev/null | grep -q "^tomcat.service"; then
    systemctl stop tomcat 2>/dev/null || true
fi

# ------------------------------------------------------------
# Create Tomcat user
# ------------------------------------------------------------

log "Creating Tomcat user"

if id "${TOMCAT_USER}" >/dev/null 2>&1; then

    echo "User '${TOMCAT_USER}' already exists."

else

    useradd \
        --system \
        --home-dir "${TOMCAT_HOME}" \
        --create-home \
        --shell /usr/sbin/nologin \
        "${TOMCAT_USER}"

    echo "Created user '${TOMCAT_USER}'."

fi

# ------------------------------------------------------------
# Backup existing Tomcat installation
# ------------------------------------------------------------

if [[ -d "${TOMCAT_HOME}" ]]; then

    log "Backing up existing Tomcat installation"

    BACKUP_DIR="/opt/tomcat-backup-$(date +%Y%m%d-%H%M%S)"

    echo "Moving:"
    echo "${TOMCAT_HOME}"
    echo
    echo "To:"
    echo "${BACKUP_DIR}"

    mv "${TOMCAT_HOME}" "${BACKUP_DIR}"

fi

# ------------------------------------------------------------
# Install Tomcat
# ------------------------------------------------------------

log "Installing Tomcat"

mkdir -p "${TOMCAT_HOME}"

tar \
    -xzf "${TOMCAT_TARBALL}" \
    -C "${TOMCAT_HOME}" \
    --strip-components=1

# ------------------------------------------------------------
# Configure permissions
# ------------------------------------------------------------

log "Configuring Tomcat permissions"

chown -R "${TOMCAT_USER}:${TOMCAT_GROUP}" "${TOMCAT_HOME}"

chmod +x "${TOMCAT_HOME}"/bin/*.sh

mkdir -p \
    "${TOMCAT_HOME}/logs" \
    "${TOMCAT_HOME}/temp" \
    "${TOMCAT_HOME}/work"

chown -R "${TOMCAT_USER}:${TOMCAT_GROUP}" \
    "${TOMCAT_HOME}/logs" \
    "${TOMCAT_HOME}/temp" \
    "${TOMCAT_HOME}/work"

# ------------------------------------------------------------
# Configure Tomcat users
# ------------------------------------------------------------

log "Configuring Tomcat Manager users"

TOMCAT_USERS_FILE="${TOMCAT_HOME}/conf/tomcat-users.xml"

if [[ -f "${TOMCAT_USERS_FILE}" ]]; then

    cp \
        "${TOMCAT_USERS_FILE}" \
        "${TOMCAT_USERS_FILE}.original"

else

    cat > "${TOMCAT_USERS_FILE}" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users>
</tomcat-users>
EOF

fi

# Remove our previous generated block if present
sed -i \
    '/<!-- BEGIN ONE-CLICK-TOMCAT-USERS -->/,/<!-- END ONE-CLICK-TOMCAT-USERS -->/d' \
    "${TOMCAT_USERS_FILE}"

# Add users
sed -i "/<\/tomcat-users>/i\\
    <!-- BEGIN ONE-CLICK-TOMCAT-USERS -->\\
    <role rolename=\"manager-gui\"/>\\
    <role rolename=\"admin-gui\"/>\\
    <user username=\"${MANAGER_USERNAME}\" password=\"${MANAGER_PASSWORD}\" roles=\"manager-gui\"/>\\
    <user username=\"${ADMIN_USERNAME}\" password=\"${ADMIN_PASSWORD}\" roles=\"manager-gui,admin-gui\"/>\\
    <!-- END ONE-CLICK-TOMCAT-USERS -->" \
    "${TOMCAT_USERS_FILE}"

chown "${TOMCAT_USER}:${TOMCAT_GROUP}" \
    "${TOMCAT_USERS_FILE}"

chmod 640 "${TOMCAT_USERS_FILE}"

# ------------------------------------------------------------
# Configure Manager
# ------------------------------------------------------------

log "Configuring Tomcat Manager"

MANAGER_CONTEXT="${TOMCAT_HOME}/webapps/manager/META-INF/context.xml"

if [[ -f "${MANAGER_CONTEXT}" ]]; then

    cp \
        "${MANAGER_CONTEXT}" \
        "${MANAGER_CONTEXT}.original"

    # Remove the default localhost-only RemoteAddrValve.
    # This allows remote Manager access.
    #
    # IMPORTANT:
    # In production, restrict port 8080 using AWS Security Groups,
    # firewall rules or a reverse proxy.

    sed -i \
        '/org\.apache\.catalina\.valves\.RemoteAddrValve/{
            s/^/<!-- /
            s#/>#/> -->/
        }' \
        "${MANAGER_CONTEXT}" || true

    chown "${TOMCAT_USER}:${TOMCAT_GROUP}" \
        "${MANAGER_CONTEXT}"

fi

# ------------------------------------------------------------
# Configure Host Manager
# ------------------------------------------------------------

log "Configuring Host Manager"

HOST_MANAGER_CONTEXT="${TOMCAT_HOME}/webapps/host-manager/META-INF/context.xml"

if [[ -f "${HOST_MANAGER_CONTEXT}" ]]; then

    cp \
        "${HOST_MANAGER_CONTEXT}" \
        "${HOST_MANAGER_CONTEXT}.original"

    sed -i \
        '/org\.apache\.catalina\.valves\.RemoteAddrValve/{
            s/^/<!-- /
            s#/>#/> -->/
        }' \
        "${HOST_MANAGER_CONTEXT}" || true

    chown "${TOMCAT_USER}:${TOMCAT_GROUP}" \
        "${HOST_MANAGER_CONTEXT}"

fi

# ------------------------------------------------------------
# Create systemd service
# ------------------------------------------------------------

log "Creating systemd service"

cat > /etc/systemd/system/tomcat.service <<EOF
[Unit]
Description=Apache Tomcat ${TOMCAT_VERSION}
Documentation=https://tomcat.apache.org/tomcat-10.1-doc/
After=network.target

[Service]
Type=forking

User=${TOMCAT_USER}
Group=${TOMCAT_GROUP}

Environment="JAVA_HOME=${JAVA_HOME}"
Environment="CATALINA_HOME=${TOMCAT_HOME}"
Environment="CATALINA_BASE=${TOMCAT_HOME}"
Environment="CATALINA_PID=${TOMCAT_HOME}/temp/tomcat.pid"

Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=${TOMCAT_HOME}/bin/startup.sh
ExecStop=${TOMCAT_HOME}/bin/shutdown.sh

Restart=on-failure
RestartSec=10

UMask=0027

[Install]
WantedBy=multi-user.target
EOF

chmod 644 /etc/systemd/system/tomcat.service

# ------------------------------------------------------------
# Reload systemd
# ------------------------------------------------------------

log "Reloading systemd"

systemctl daemon-reload

# ------------------------------------------------------------
# Enable Tomcat
# ------------------------------------------------------------

log "Enabling Tomcat"

systemctl enable tomcat

# ------------------------------------------------------------
# Start Tomcat
# ------------------------------------------------------------

log "Starting Tomcat"

systemctl restart tomcat

# ------------------------------------------------------------
# Wait for Tomcat
# ------------------------------------------------------------

log "Waiting for Tomcat"

MAX_ATTEMPTS=30
ATTEMPT=1

while [[ "${ATTEMPT}" -le "${MAX_ATTEMPTS}" ]]; do

    if curl \
        --silent \
        --output /dev/null \
        --connect-timeout 2 \
        "http://127.0.0.1:${TOMCAT_PORT}/"; then

        echo
        echo "Tomcat is responding on port ${TOMCAT_PORT}."
        break

    fi

    echo "Waiting for Tomcat... ${ATTEMPT}/${MAX_ATTEMPTS}"

    sleep 2

    ATTEMPT=$((ATTEMPT + 1))

done

if [[ "${ATTEMPT}" -gt "${MAX_ATTEMPTS}" ]]; then

    echo
    echo "Tomcat did not respond."
    echo
    echo "Systemd status:"
    systemctl status tomcat --no-pager || true

    echo
    echo "Recent Tomcat logs:"
    journalctl -u tomcat --no-pager -n 50 || true

    exit 1

fi

# ------------------------------------------------------------
# Configure UFW
# ------------------------------------------------------------

log "Configuring firewall"

if command -v ufw >/dev/null 2>&1; then

    # Keep SSH access available
    ufw allow OpenSSH >/dev/null 2>&1 || true

    # Tomcat
    ufw allow "${TOMCAT_PORT}/tcp" >/dev/null 2>&1 || true

    echo "UFW rule added for port ${TOMCAT_PORT}."

fi

# ------------------------------------------------------------
# Verify Tomcat service
# ------------------------------------------------------------

log "Verifying Tomcat service"

if systemctl is-active --quiet tomcat; then

    echo "Tomcat service is RUNNING."

else

    echo "Tomcat service FAILED."

    systemctl status tomcat --no-pager || true

    exit 1

fi

# ------------------------------------------------------------
# Detect server IP
# ------------------------------------------------------------

SERVER_IP="$(hostname -I 2>/dev/null | awk '{print $1}' || true)"

if [[ -z "${SERVER_IP}" ]]; then
    SERVER_IP="YOUR_SERVER_IP"
fi

# ------------------------------------------------------------
# Final output
# ------------------------------------------------------------

log "Tomcat Installation Completed Successfully"

echo
echo "============================================================"
echo "             TOMCAT INSTALLATION DETAILS"
echo "============================================================"
echo
echo "Tomcat Version : ${TOMCAT_VERSION}"
echo "Tomcat Home    : ${TOMCAT_HOME}"
echo "Tomcat User    : ${TOMCAT_USER}"
echo "Java Home      : ${JAVA_HOME}"
echo "Tomcat Port    : ${TOMCAT_PORT}"
echo
echo "------------------------------------------------------------"
echo "Tomcat URL"
echo "------------------------------------------------------------"
echo
echo "http://${SERVER_IP}:${TOMCAT_PORT}/"
echo
echo "------------------------------------------------------------"
echo "Tomcat Manager"
echo "------------------------------------------------------------"
echo
echo "http://${SERVER_IP}:${TOMCAT_PORT}/manager/html"
echo
echo "Username: ${MANAGER_USERNAME}"
echo "Password: ${MANAGER_PASSWORD}"
echo
echo "------------------------------------------------------------"
echo "Tomcat Host Manager"
echo "------------------------------------------------------------"
echo
echo "http://${SERVER_IP}:${TOMCAT_PORT}/host-manager/html"
echo
echo "Username: ${ADMIN_USERNAME}"
echo "Password: ${ADMIN_PASSWORD}"
echo
echo "------------------------------------------------------------"
echo "Service Commands"
echo "------------------------------------------------------------"
echo
echo "Start   : systemctl start tomcat"
echo "Stop    : systemctl stop tomcat"
echo "Restart : systemctl restart tomcat"
echo "Status  : systemctl status tomcat"
echo
echo "Logs:"
echo "journalctl -u tomcat -f"
echo
echo "Tomcat log:"
echo "${TOMCAT_HOME}/logs/catalina.out"
echo
echo "============================================================"
echo "                    INSTALLATION DONE"
echo "============================================================"
echo
```
run the command 

```bash
sudo ./install-tomcat.sh 'Manager@12345' 'Admin@12345'
```

# 2. Manual 
## How To Install Apache Tomcat 10 on Ubuntu 20.04

**Java | Ubuntu 20.04 | Apache**  
**Authors:** Siddesh P M

---

## Introduction

Apache Tomcat is a web server and servlet container that is used to serve Java applications. It’s an open-source implementation of the Jakarta Servlet, Jakarta Server Pages, and other technologies of the Jakarta EE platform.

In this tutorial, you’ll deploy Apache Tomcat 10 on Ubuntu 20.04. You will install Tomcat 10, set up users and roles, and navigate the admin user interface.

---

## Prerequisites

- One Ubuntu 20.04 server with a sudo non-root user and a firewall, which you can set up by following the Ubuntu 20.04 Initial Server Setup.

---

## Step 1 — Installing Tomcat

In this section, you will set up Tomcat 10 on your server. To begin, you will download its latest version and set up a separate user and appropriate permissions for it. You will also install the Java Development Kit (JDK).

For security purposes, Tomcat should run under a separate, unprivileged user. Run the following command to create a user called `tomcat`:

```bash
sudo useradd -m -d /opt/tomcat -U -s /bin/false tomcat
```

By supplying `/bin/false` as the user’s default shell, you ensure that it’s not possible to log in as `tomcat`.

### Install the JDK

First, update the package manager cache:

```bash
sudo apt update
```

Then, install the JDK:

```bash
sudo apt install default-jdk
```

Answer `y` when prompted to continue with the installation.

Verify the Java installation:

```bash
java -version
```

**Expected Output:**

```bash
openjdk version "11.0.14" 2022-01-18
OpenJDK Runtime Environment (build 11.0.14+9-Ubuntu-0ubuntu2.20.04)
OpenJDK 64-Bit Server VM (build 11.0.14+9-Ubuntu-0ubuntu2.20.04, mixed mode, sharing)
```

### Download and Install Tomcat

Navigate to the `/tmp` directory:

```bash
cd /tmp
```

Download the latest Tomcat 10 Core Linux build:

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.34/bin/apache-tomcat-10.1.34.tar.gz
```

Extract the archive:

```bash
sudo tar xzvf apache-tomcat-10*tar.gz -C /opt/tomcat --strip-components=1
```

Set ownership and permissions:

```bash
sudo chown -R tomcat:tomcat /opt/tomcat/
sudo chmod -R u+x /opt/tomcat/bin
```

---

## Step 2 — Configuring Admin Users

Edit the Tomcat users configuration file:

```bash
sudo nano /opt/tomcat/conf/tomcat-users.xml
```

Add the following lines before the closing `</tomcat-users>` tag:

```xml
<role rolename="manager-gui" />
<user username="manager" password="manager_password" roles="manager-gui" />

<role rolename="admin-gui" />
<user username="admin" password="admin_password" roles="manager-gui,admin-gui" />
```

Replace `manager_password` and `admin_password` with your own values. Save and close the file.

To remove access restrictions, edit the `context.xml` file for the Manager app:

```bash
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
```

Comment out the following line:

```xml
<!--  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\..*" /> -->
```

Repeat this step for the Host Manager:

```bash
sudo nano /opt/tomcat/webapps/host-manager/META-INF/context.xml
```

---

## Step 3 — Creating a systemd Service

Find the Java installation path:

```bash
sudo update-java-alternatives -l
```

Expected Output:

```bash
java-1.11.0-openjdk-amd64      1111       /usr/lib/jvm/java-1.11.0-openjdk-amd64
```

Create a systemd service file:

```bash
sudo nano /etc/systemd/system/tomcat.service
```

Add the following content:

```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

Save and close the file.

Reload the systemd daemon:

```bash
sudo systemctl daemon-reload
```

Start Tomcat:

```bash
sudo systemctl start tomcat
```

Check the status:

```bash
sudo systemctl status tomcat
```

Enable Tomcat to start on boot:

```bash
sudo systemctl enable tomcat
```

---

## Step 4 — Accessing the Web Interface

Allow traffic to Tomcat’s port:

```bash
sudo ufw allow 8080
```

Access Tomcat in your browser:

```
http://your_server_ip:8080
```

Log in using the credentials you set in Step 2.

### Manager App:

Here, you can start, stop, reload, deploy, and undeploy Java applications.

### Host Manager:

Use this interface to manage virtual hosts.

---

## Conclusion

In this tutorial, you installed Apache Tomcat 10 on Ubuntu 20.04, created an admin user, configured a systemd service, and accessed the Tomcat web interface.

You’re now ready to deploy Java applications on your Tomcat server!
