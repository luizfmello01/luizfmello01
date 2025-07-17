# Nginx Container Setup for Network Testing

## Step 1: Pull Nginx Image

```bash
# Pull the official Nginx image (version 1.28 with Alpine)
docker pull nginx:1.28-alpine

# Verify the image is downloaded
docker images | grep nginx
```

## Step 2: Create a Simple Test HTML Page

```bash
# Create a directory for Nginx content
mkdir -p ~/nginx-test

# Create a simple test HTML file
cat > ~/nginx-test/index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Docker Nginx Test</title>
    <style>
        body { 
            font-family: Arial, sans-serif; 
            text-align: center; 
            margin-top: 100px;
            background-color: #f0f0f0;
        }
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            display: inline-block;
        }
        h1 { color: #333; }
        p { color: #666; }
        .success { color: #28a745; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Nginx Container Test</h1>
        <p class="success">✅ Container is running successfully!</p>
        <p>Server: $(hostname)</p>
        <p>Date: $(date)</p>
        <p>If you can see this page, your Docker Nginx container is working correctly.</p>
    </div>
</body>
</html>
EOF
```

## Step 3: Run Nginx Container

### Option A: Simple Run (Basic)
```bash
# Run Nginx container with port mapping
docker run -d \
  --name nginx-test \
  -p 80:80 \
  -v ~/nginx-test:/usr/share/nginx/html:ro \
  nginx:1.28-alpine

# Check if container is running
docker ps
```

### Option B: Run with Custom Port (Alternative)
```bash
# Run on custom port (e.g., 8080) if port 80 is busy
docker run -d \
  --name nginx-test \
  -p 8080:80 \
  -v ~/nginx-test:/usr/share/nginx/html:ro \
  nginx:1.28-alpine
```

## Step 4: Test the Container

### Local Testing
```bash
# Test from the server itself
curl http://localhost

# Or if using custom port
curl http://localhost:8080

# Check container logs
docker logs nginx-test

# Check container status
docker ps | grep nginx-test
```

### External Testing
```bash
# Get server IP address
ip addr show | grep inet

# Test from external machine (replace SERVER_IP with actual IP)
# curl http://SERVER_IP
# Or open in browser: http://SERVER_IP
```

## Step 5: Firewall Configuration (if needed)

### For UFW (Ubuntu Firewall)
```bash
# Check firewall status
sudo ufw status

# Allow HTTP traffic if firewall is active
sudo ufw allow 80/tcp

# Or for custom port
sudo ufw allow 8080/tcp
```

### For iptables (if used)
```bash
# Allow HTTP traffic
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Or for custom port
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

## Step 6: Management Commands

```bash
# Stop the container
docker stop nginx-test

# Start the container
docker start nginx-test

# Restart the container
docker restart nginx-test

# Remove the container
docker rm nginx-test

# Remove the container (force)
docker rm -f nginx-test

# View container details
docker inspect nginx-test

# Execute commands inside container (Alpine uses sh instead of bash)
docker exec -it nginx-test /bin/sh
```

## Step 7: Verify External Access

### Check from outside the server:
1. **Browser**: Open `http://YOUR_SERVER_IP` in a web browser
2. **Command line**: `curl http://YOUR_SERVER_IP`
3. **Mobile/Other device**: Try accessing from different networks

### Expected Result:
You should see the test HTML page with "Nginx Container Test" message.

## Troubleshooting

### Container not starting:
```bash
# Check container logs
docker logs nginx-test

# Check if port is already in use
sudo netstat -tulpn | grep :80
```

### Cannot access from external network:
```bash
# Check if container is running
docker ps

# Check firewall rules
sudo ufw status
sudo iptables -L

# Check if Docker is exposing the port
docker port nginx-test

# Test local connectivity first
curl http://localhost
```

### Port already in use:
```bash
# Use a different port
docker run -d --name nginx-test -p 8080:80 -v ~/nginx-test:/usr/share/nginx/html:ro nginx:1.28-alpine
```

## Clean Up (Optional)

```bash
# Stop and remove container
docker stop nginx-test
docker rm nginx-test

# Remove custom HTML files
rm -rf ~/nginx-test

# Remove Nginx image (optional)
docker rmi nginx:1.28-alpine
```

That's it! Your Nginx container should now be accessible from external networks for testing connectivity.