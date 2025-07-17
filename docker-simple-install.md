# Simple Docker Installation Guide (Testing Only)

## Prerequisites
- Ubuntu/Debian server
- Root or sudo access
- Internet connectivity

## Step 1: Update System

```bash
sudo apt update
sudo apt upgrade -y
```

## Step 2: Install Docker

```bash
# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## Step 3: Start Docker Service

```bash
# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker
```

## Step 4: Add User to Docker Group (Optional)

```bash
# Add current user to docker group to run without sudo
sudo usermod -aG docker $USER

# Note: Log out and back in for this to take effect
```

## Step 5: Test Installation

```bash
# Test Docker installation
sudo docker run hello-world

# Check Docker version
sudo docker --version

# Check Docker Compose version
sudo docker compose version
```

## Basic Usage Commands

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# List images
docker images

# Remove container
docker rm <container_name>

# Remove image
docker rmi <image_name>

# View container logs
docker logs <container_name>
```

That's it! Docker is now installed and ready for testing.