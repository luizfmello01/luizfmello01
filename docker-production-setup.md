# Docker Production Server Setup Guide

## Prerequisites
- Ubuntu/Debian-based server (adjust commands for CentOS/RHEL if needed)
- Root or sudo access
- Internet connectivity
- At least 2GB RAM and 20GB disk space

## Step 1: Update System Packages

```bash
# Update package index
sudo apt update

# Upgrade existing packages
sudo apt upgrade -y

# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

## Step 2: Add Docker's Official GPG Key

```bash
# Create directory for keyrings
sudo mkdir -p /etc/apt/keyrings

# Download and add Docker's GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set proper permissions
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

## Step 3: Add Docker Repository

```bash
# Add Docker repository to sources list
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index again
sudo apt update
```

## Step 4: Install Docker Engine

```bash
# Install Docker CE, CLI, and containerd
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
sudo docker --version
sudo docker compose version
```

## Step 5: Configure Docker for Production

### 5.1 Create Docker Daemon Configuration

```bash
# Create Docker daemon configuration directory
sudo mkdir -p /etc/docker

# Create daemon.json configuration file
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "seccomp-profile": "/etc/docker/seccomp.json",
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  }
}
EOF
```

### 5.2 Configure Docker Service

```bash
# Enable Docker service to start on boot
sudo systemctl enable docker

# Start Docker service
sudo systemctl start docker

# Verify Docker is running
sudo systemctl status docker
```

## Step 6: Security Configuration

### 6.1 Create Docker User Group (Optional)

```bash
# Create docker group (usually already exists)
sudo groupadd docker

# Add your user to docker group (replace 'username' with actual username)
sudo usermod -aG docker $USER

# Note: You'll need to log out and back in for group changes to take effect
```

### 6.2 Configure Firewall (if UFW is used)

```bash
# Check if UFW is active
sudo ufw status

# If UFW is active, configure Docker rules
sudo ufw allow 2376/tcp comment "Docker daemon TLS"
sudo ufw allow 2377/tcp comment "Docker swarm management"
sudo ufw allow 7946/tcp comment "Docker swarm node communication"
sudo ufw allow 7946/udp comment "Docker swarm node communication"
sudo ufw allow 4789/udp comment "Docker overlay network"
```

### 6.3 Configure Docker Socket Security

```bash
# Set proper permissions on Docker socket
sudo chmod 660 /var/run/docker.sock
sudo chown root:docker /var/run/docker.sock
```

## Step 7: Configure Resource Limits

### 7.1 Set System Limits

```bash
# Edit limits configuration
sudo tee -a /etc/security/limits.conf > /dev/null <<EOF
# Docker resource limits
* soft nofile 65536
* hard nofile 65536
* soft nproc 32768
* hard nproc 32768
EOF
```

### 7.2 Configure Systemd Service Limits

```bash
# Create systemd override directory
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create override configuration
sudo tee /etc/systemd/system/docker.service.d/override.conf > /dev/null <<EOF
[Service]
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TasksMax=infinity
EOF
```

## Step 8: Configure Log Rotation

```bash
# Create logrotate configuration for Docker
sudo tee /etc/logrotate.d/docker > /dev/null <<EOF
/var/lib/docker/containers/*/*.log {
    rotate 7
    daily
    compress
    size=1M
    missingok
    delaycompress
    copytruncate
}
EOF
```

## Step 9: Restart and Verify Docker

```bash
# Reload systemd configuration
sudo systemctl daemon-reload

# Restart Docker service
sudo systemctl restart docker

# Verify Docker is running correctly
sudo docker info

# Test Docker installation
sudo docker run hello-world
```

## Step 10: Production Security Hardening

### 10.1 Disable Docker API Access (if not needed)

```bash
# Ensure Docker daemon only listens on Unix socket
# Verify in daemon.json that no "hosts" configuration exposes TCP ports
grep -v "hosts" /etc/docker/daemon.json
```

### 10.2 Enable Content Trust (optional)

```bash
# Set environment variable for content trust
echo 'export DOCKER_CONTENT_TRUST=1' | sudo tee -a /etc/environment
```

### 10.3 Configure AppArmor/SELinux (if applicable)

```bash
# For Ubuntu/Debian with AppArmor
sudo aa-status | grep docker

# For CentOS/RHEL with SELinux
# sestatus
```

## Step 11: Monitoring and Maintenance

### 11.1 Set up basic monitoring

```bash
# Create a script to monitor Docker health
sudo tee /usr/local/bin/docker-health-check.sh > /dev/null <<EOF
#!/bin/bash
# Basic Docker health check
docker system df
docker system events --filter type=container --since 1h --until now
EOF

sudo chmod +x /usr/local/bin/docker-health-check.sh
```

### 11.2 Configure automatic cleanup

```bash
# Create cleanup script
sudo tee /usr/local/bin/docker-cleanup.sh > /dev/null <<EOF
#!/bin/bash
# Docker cleanup script
docker system prune -f
docker volume prune -f
docker image prune -a -f
EOF

sudo chmod +x /usr/local/bin/docker-cleanup.sh

# Add to crontab for weekly cleanup
echo "0 2 * * 0 /usr/local/bin/docker-cleanup.sh" | sudo crontab -
```

## Step 12: Backup Configuration

```bash
# Create backup of Docker configuration
sudo mkdir -p /opt/docker-backup
sudo cp -r /etc/docker /opt/docker-backup/
sudo cp /etc/systemd/system/docker.service.d/override.conf /opt/docker-backup/ 2>/dev/null || true
```

## Verification Commands

After completing the installation, run these commands to verify everything is working:

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Check Docker system information
docker system info

# Check Docker service status
systemctl status docker

# Test Docker functionality
docker run --rm hello-world

# Check Docker networks
docker network ls

# Check Docker volumes
docker volume ls

# Check system resources
docker system df
```

## Troubleshooting

### Common Issues:

1. **Permission denied errors**: Ensure your user is in the docker group and you've logged out/in
2. **Service fails to start**: Check logs with `sudo journalctl -u docker.service`
3. **Network issues**: Verify firewall settings and DNS configuration
4. **Storage issues**: Monitor disk space with `docker system df`

### Log Locations:
- Docker daemon logs: `sudo journalctl -u docker.service`
- Container logs: `docker logs <container_name>`
- System logs: `/var/log/syslog` or `/var/log/messages`

## Next Steps

After Docker is installed and configured:
1. Set up Docker registry authentication if using private registries
2. Configure container orchestration (Docker Swarm or Kubernetes)
3. Implement backup strategies for volumes and images
4. Set up monitoring with tools like Prometheus and Grafana
5. Configure CI/CD pipelines for automated deployments

## Security Considerations

- Regularly update Docker and host system
- Use official images when possible
- Scan images for vulnerabilities
- Implement proper network segmentation
- Use secrets management for sensitive data
- Enable audit logging
- Regularly review and update security policies