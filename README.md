# Setting Up Red Hat Quay POC Registry Using Podman in Rootless Mode on RHEL 9

## Why Quay?

Red Hat Quay is a powerful, open-source container registry with a robust solution for storing, building, and distributing container images. Whether you're setting up a home lab, testing in an enterprise environment, or developing applications, Quay offers several compelling advantages:

- **Fully open-source**: The Project Quay upstream version is entirely open source, making it perfect for home labs and testing environments
- **Enterprise-grade security**: Built-in vulnerability scanning, image signing, and access controls
- **Powerful build system**: Automated container builds triggered by git commits
- **Efficient storage**: Configurable storage backends with content deduplication
- **Rich API support**: Comprehensive API for automation and integration
- **High availability**: Designed for redundancy and horizontal scaling (in production deployments)

Quay is an excellent choice for developers and organizations who need more control and features than basic registries provide, without the overhead of commercial solutions.

> **Note:** This guide is only for Proof of Concept (POC) deployments. It is not intended for production use.

## Prerequisites

- RHEL 9 system with at least:
  - 10 GB disk space for the operating system
  - 10 GB disk space for container storage
  - 10 GB disk space for Quay local storage
  - 4 GB or more of memory
- Valid Red Hat subscription
- Internet connectivity to pull container images

## Step 1: System Preparation and Registration

Register and subscribe your RHEL 9 system:

```bash
# Register your system with your Red Hat account
sudo subscription-manager register --username=<your-username>

# List available subscriptions
sudo subscription-manager list --available

# Attach to a subscription pool (replace POOL_ID with your actual pool ID)
sudo subscription-manager attach --pool=POOL_ID

# Update your system
sudo dnf update -y
```

## Step 2: Install Podman

Install Podman and the required packages for rootless mode:

```bash
# Install podman and rootless dependencies
sudo dnf install -y podman slirp4netns fuse-overlayfs
```

Alternatively, you can install the container-tools module:

```bash
# Install container-tools module
sudo dnf module install -y container-tools
```

Verify Podman installation:

```bash
podman --version
```

## Step 3: Authenticate with Red Hat Registry

Set up authentication to registry.redhat.io:

```bash
# Log in to the Red Hat Registry
podman login registry.redhat.io
```

Enter your Red Hat account credentials when prompted.

## Step 4: Create Directory Structure and Network

Set up directories for Quay POC deployment:

```bash
# Create all required directories in one command
mkdir -p ~/quay/{postgres-quay,storage,config}
```

Create a dedicated Podman network for container communication:

```bash
# Create a podman network for the Quay components
podman network create quay-network

# Verify the network was created
podman network ls
```

## Step 5: Deploy the PostgreSQL Database

Create a podman volume for the PostgreSQL data:

```bash
# Create a named volume for PostgreSQL data
podman volume create postgres-quay-data
```

Run the PostgreSQL container using the named volume and the quay-network:

```bash
# Run PostgreSQL container with a named volume and on the quay-network
podman run -d --name postgresql-quay \
  --network quay-network \
  -e POSTGRESQL_USER=quayuser \
  -e POSTGRESQL_PASSWORD=quaypass \
  -e POSTGRESQL_DATABASE=quay \
  -e POSTGRESQL_ADMIN_PASSWORD=adminpass \
  -p 5432:5432 \
  -v postgres-quay-data:/var/lib/pgsql/data \
  registry.redhat.io/rhel9/postgresql-16
```

Verify that the container is running:

```bash
podman ps
```

Install the required pg_trgm extension:

```bash
podman exec -it postgresql-quay /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | psql -d quay -U postgres'
```

## Step 6: Deploy Redis

Run Redis container on the quay-network:

```bash
# Run Redis container on the quay-network
podman run -d --name redis \
  --network quay-network \
  -p 6379:6379 \
  -e REDIS_PASSWORD=strongpassword \
  registry.redhat.io/rhel8/redis-6
```

Verify that the Redis container is running:

```bash
podman ps
```

## Step 7: Determine Container Network Configuration

Get information about the containers and their IP addresses:

```bash
# Get the IP addresses of the postgresql-quay and redis containers
podman inspect -f "{{.NetworkSettings.Networks}}" postgresql-quay
podman inspect -f "{{.NetworkSettings.Networks}}" redis
```

Store the server's external IP for use in the Quay configuration:

```bash
# Get your server's external IP for Quay access
export SERVER_IP=$(ip -4 addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | grep -v "127.0.0.1" | head -n 1)
echo "Server external IP: $SERVER_IP"
```

Configure firewall rules if needed:

```bash
# Open required ports in the firewall (requires root privileges)
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --reload
```

## Step 8: Launch Quay Configuration Tool

Run the Quay configuration tool container:

```bash
# Run the Quay configuration container on the quay-network
podman run --rm -it --name quay_config \
  --network quay-network \
  -p 8080:8080 \
  -p 8443:8443 \
  registry.redhat.io/quay/quay-rhel8:v3.13.3 config secret
```

Access the Quay configuration UI at:

```
http://$SERVER_IP:8080
```

The default credentials are:
- Username: quayconfig
- Password: secret

## Step 9: Configure Quay

In the Quay configuration web UI, configure the following:

### 1. Basic Configuration
- Server Hostname: Enter your server IP address (the same defined in the previous step)
- Registry Title: Enter a title for your registry (e.g., "Quay POC Registry")

### 2. Database Configuration
- Database Type: PostgreSQL
- Database Server: postgresql-quay (container name in the network)
- Database Name: quay
- Username: quayuser
- Password: quaypass
- Port: 5432

### 3. Redis Configuration
- Redis Hostname: redis (container name in the network)
- Redis Port: 6379
- Redis Password: strongpassword

### 4. Storage Configuration
- Storage Type: Local Storage
- Path: /datastorage/registry

### 5. Security Scanner
- You can skip this for a basic POC deployment

After completing the configuration, click "Validate Configuration" and then "Download Configuration".

## Step 10: Deploy Quay Registry

Run the Quay registry container:

```bash
# Extract the configuration to your config directory
tar -xvf quay-config.tar.gz -C ~/quay/config

# Run the Quay registry container on the quay-network
podman run -d --name quay \
  --network quay-network \
  -p 8080:8080 \
  -p 8443:8443 \
  -v ~/quay/config:/conf/stack \
  -v ~/quay/storage:/datastorage \
  registry.redhat.io/quay/quay-rhel8:v3.13.3
```

Alternative: Running Quay with a manual configuration file:

If you're still experiencing connection issues, you can configure Quay using a manually created config.yaml file:

```bash
# Create a minimal config.yaml file
mkdir -p ~/quay/config
cat > ~/quay/config/config.yaml << EOF
FEATURE_SECURITY_SCANNER: false
SERVER_HOSTNAME: $SERVER_IP
PREFERRED_URL_SCHEME: http
DATABASE_SECRET_KEY: e9bd34f4-900c-436a-979e-7530e5d74ac8
SECRET_KEY: e9bd34f4-900c-436a-979e-7530e5d74ac8
SETUP_COMPLETE: true
DB_URI: postgresql://quayuser:quaypass@postgresql-quay:5432/quay
BUILDLOGS_REDIS: {"host": "redis", "port": 6379, "password": "strongpassword"}
USER_EVENTS_REDIS: {"host": "redis", "port": 6379, "password": "strongpassword"}
DISTRIBUTED_STORAGE_CONFIG:
  default:
    - LocalStorage
    - storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE: [default]
EOF

# Run the Quay registry container with the manual config
podman run -d --name quay \
  --network quay-network \
  -p 8080:8080 \
  -p 8443:8443 \
  -v ~/quay/config:/conf/stack \
  -v ~/quay/storage:/datastorage \
  registry.redhat.io/quay/quay-rhel8:v3.13.3
```

Verify that the Quay container is running:

```bash
podman ps
```

## Step 11: Access and Test Your Quay Registry

Access your Quay registry at:

```
http://$SERVER_IP:8080
```

Create a new user:
1. Click on "Create Account" on the login page
2. Fill in the required information
3. Create a new organization (optional)
4. Create a new repository

Test pushing an image:

```bash
# Pull a test image
podman pull docker.io/library/busybox:latest

# Tag the image for your registry
podman tag docker.io/library/busybox:latest $SERVER_IP:8080/yourusername/busybox:test

# Push the image to your registry
podman push --tls-verify=false $SERVER_IP:8080/yourusername/busybox:test
```

## Optional: Make Containers Start on Boot

For rootless Podman, create systemd user services:

```bash
# Create systemd user service directories
mkdir -p ~/.config/systemd/user

# Generate service files
cd ~/.config/systemd/user
podman generate systemd --name postgresql-quay --files --new
podman generate systemd --name redis --files --new
podman generate systemd --name quay --files --new

# Enable services
systemctl --user enable container-postgresql-quay.service
systemctl --user enable container-redis.service
systemctl --user enable container-quay.service

# Enable lingering for your user to allow services to run without being logged in
sudo loginctl enable-linger $(whoami)
```

## Troubleshooting

- If you encounter permission issues, check that your user has the proper permissions for Podman volumes.
- For networking issues, ensure the ports are accessible and not being blocked by firewall rules.
- If containers fail to start, check container logs using `podman logs [container-name]`.
- For database connection issues, verify your PostgreSQL container is running and accessible.

## Additional Resources

- [Red Hat Quay 3.13 Documentation - Proof of Concept Deployment](https://docs.redhat.com/en/documentation/red_hat_quay/3.13/html-single/proof_of_concept_-_deploying_red_hat_quay/index)
- [RHEL 9 - Building, Running, and Managing Containers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/assembly_starting-with-containers_building-running-and-managing-containers)
- [Podman Documentation](https://podman.io/docs)

## Warning

This is a Proof of Concept deployment for testing purposes only. It uses local storage, which is not guaranteed to provide the required read-after-write consistency and data integrity guarantees during parallel access that a registry requires. Do not use this deployment type for production.
