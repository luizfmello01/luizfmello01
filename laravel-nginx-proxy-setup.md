# Laravel + Nginx Reverse Proxy Setup

## Overview
This setup creates two containers:
- **Nginx Container**: Acts as reverse proxy (port 80)
- **Laravel Container**: PHP application with Laravel (internal port 9000)

## Prerequisites
- Docker and Docker Compose installed
- Basic understanding of Docker networking

## Step 1: Create Project Structure

```bash
# Create project directory
mkdir ~/laravel-nginx-setup
cd ~/laravel-nginx-setup

# Create directory structure
mkdir -p nginx/conf
mkdir -p laravel/app
mkdir -p laravel/logs
```

## Step 2: Create Laravel Application

### Option A: Create New Laravel Project
```bash
# Create a simple Laravel application structure
cd ~/laravel-nginx-setup/laravel/app

# Create basic Laravel files for testing
cat > index.php << 'EOF'
<?php
// Simple PHP info page for testing
phpinfo();

// Laravel-style response
echo "<hr><h2>Laravel Container Test</h2>";
echo "<p><strong>Status:</strong> ✅ Laravel container is running!</p>";
echo "<p><strong>Server:</strong> " . gethostname() . "</p>";
echo "<p><strong>PHP Version:</strong> " . phpversion() . "</p>";
echo "<p><strong>Date:</strong> " . date('Y-m-d H:i:s') . "</p>";
?>
EOF

# Create a test API endpoint
mkdir -p api
cat > api/test.php << 'EOF'
<?php
header('Content-Type: application/json; charset=utf-8');
header('Access-Control-Allow-Origin: *');

$response = [
    'status' => 'success',
    'message' => 'Laravel API is working!',
    'container' => gethostname(),
    'timestamp' => date('c'),
    'php_version' => phpversion()
];

echo json_encode($response, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
?>
EOF
```

### Option B: Download Real Laravel (Alternative)
```bash
# If you want to use real Laravel, you can download it
cd ~/laravel-nginx-setup/laravel
# wget -O laravel.tar.gz https://github.com/laravel/laravel/archive/refs/heads/master.tar.gz
# tar -xzf laravel.tar.gz --strip-components=1
# rm laravel.tar.gz
```

## Step 3: Create PHP-FPM Dockerfile

```bash
# Create Dockerfile for Laravel container
cat > ~/laravel-nginx-setup/laravel/Dockerfile << 'EOF'
FROM php:8.2-fpm-alpine

# Install system dependencies
RUN apk add --no-cache \
    curl \
    libpng-dev \
    libxml2-dev \
    zip \
    unzip \
    git \
    oniguruma-dev

# Install PHP extensions
RUN docker-php-ext-install \
    pdo_mysql \
    mbstring \
    exif \
    pcntl \
    bcmath \
    gd \
    xml

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy application files
COPY ./app /var/www/html

# Set permissions
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html

# Expose port 9000 for PHP-FPM
EXPOSE 9000

# Start PHP-FPM
CMD ["php-fpm"]
EOF
```

## Step 4: Create Nginx Configuration

```bash
# Create Nginx configuration for reverse proxy
cat > ~/laravel-nginx-setup/nginx/conf/default.conf << 'EOF'
upstream laravel_backend {
    server laravel-app:9000;
}

server {
    listen 80;
    server_name localhost;
    charset utf-8;
    
    # Document root
    root /var/www/html;
    index index.php index.html;
    
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    
    # Main location block
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # PHP-FPM configuration
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass laravel_backend;
        fastcgi_index index.php;
        
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        
        # FastCGI settings
        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 60s;
        fastcgi_read_timeout 60s;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }
    
    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
    
    # Handle static files
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
}
EOF
```

## Step 5: Create Docker Compose Configuration

```bash
# Create docker-compose.yml
cat > ~/laravel-nginx-setup/docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Laravel PHP-FPM Container
  laravel-app:
    build:
      context: ./laravel
      dockerfile: Dockerfile
    container_name: laravel-app
    restart: unless-stopped
    working_dir: /var/www/html
    volumes:
      - ./laravel/app:/var/www/html
      - ./laravel/logs:/var/log
    networks:
      - laravel-network
    environment:
      - APP_ENV=local
      - APP_DEBUG=true
    
  # Nginx Reverse Proxy Container
  nginx-proxy:
    image: nginx:1.28-alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./laravel/app:/var/www/html:ro
      - ./nginx/logs:/var/log/nginx
    depends_on:
      - laravel-app
    networks:
      - laravel-network

networks:
  laravel-network:
    driver: bridge
    
volumes:
  laravel-data:
EOF
```

## Step 6: Create Nginx Logs Directory

```bash
# Create logs directories
mkdir -p ~/laravel-nginx-setup/nginx/logs
mkdir -p ~/laravel-nginx-setup/laravel/logs

# Set permissions
chmod 755 ~/laravel-nginx-setup/nginx/logs
chmod 755 ~/laravel-nginx-setup/laravel/logs
```

## Step 7: Build and Start Containers

```bash
# Navigate to project directory
cd ~/laravel-nginx-setup

# Build and start containers
docker-compose up -d --build

# Check containers status
docker-compose ps

# View logs
docker-compose logs -f
```

## Step 8: Test the Setup

### Local Testing
```bash
# Test PHP application
curl http://localhost

# Test API endpoint
curl http://localhost/api/test.php

# Check if containers are running
docker ps

# Check network connectivity
docker exec nginx-proxy ping laravel-app
```

### Browser Testing
1. Open `http://YOUR_SERVER_IP` - Should show PHP info page
2. Open `http://YOUR_SERVER_IP/api/test.php` - Should show JSON response

## Step 9: Management Commands

```bash
# View logs
docker-compose logs nginx-proxy
docker-compose logs laravel-app

# Restart services
docker-compose restart

# Stop services
docker-compose down

# Rebuild and restart
docker-compose down
docker-compose up -d --build

# Execute commands in containers
docker exec -it laravel-app sh
docker exec -it nginx-proxy sh

# View network details
docker network ls
docker network inspect laravel-nginx-setup_laravel-network
```

## Step 10: Firewall Configuration

```bash
# Allow HTTP traffic
sudo ufw allow 80/tcp

# Allow HTTPS traffic (if needed later)
sudo ufw allow 443/tcp

# Check firewall status
sudo ufw status
```

## Step 11: Health Check Script

```bash
# Create health check script
cat > ~/laravel-nginx-setup/health-check.sh << 'EOF'
#!/bin/bash

echo "=== Laravel + Nginx Health Check ==="
echo

# Check containers
echo "Container Status:"
docker-compose ps

echo
echo "Network Test:"
# Test main page
curl -s -o /dev/null -w "Main Page: %{http_code}\n" http://localhost

# Test API
curl -s -o /dev/null -w "API Endpoint: %{http_code}\n" http://localhost/api/test.php

echo
echo "Container Communication:"
docker exec nginx-proxy ping -c 1 laravel-app > /dev/null 2>&1 && echo "✅ Nginx can reach Laravel" || echo "❌ Nginx cannot reach Laravel"

echo
echo "Disk Usage:"
docker system df

echo
echo "Recent Logs (last 5 lines):"
echo "--- Nginx Logs ---"
docker-compose logs --tail=5 nginx-proxy
echo "--- Laravel Logs ---"
docker-compose logs --tail=5 laravel-app
EOF

chmod +x ~/laravel-nginx-setup/health-check.sh
```

## Step 12: Environment Configuration (Optional)

```bash
# Create environment file for Laravel
cat > ~/laravel-nginx-setup/.env << 'EOF'
# Laravel Environment
APP_NAME="Laravel Docker Test"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

# Database (if needed later)
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=secret

# Cache
CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_CONNECTION=sync
EOF
```

## Troubleshooting

### Common Issues:

1. **502 Bad Gateway Error:**
```bash
# Check if Laravel container is running
docker-compose ps

# Check Laravel container logs
docker-compose logs laravel-app

# Verify network connectivity
docker exec nginx-proxy ping laravel-app
```

2. **Permission Denied:**
```bash
# Fix file permissions
sudo chown -R $USER:$USER ~/laravel-nginx-setup
chmod -R 755 ~/laravel-nginx-setup/laravel/app
```

3. **Container Communication Issues:**
```bash
# Check network
docker network ls
docker network inspect laravel-nginx-setup_laravel-network

# Verify container names
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

4. **Port Already in Use:**
```bash
# Check what's using port 80
sudo netstat -tulpn | grep :80

# Use different port in docker-compose.yml
# Change "80:80" to "8080:80"
```

### Log Locations:
- Nginx logs: `~/laravel-nginx-setup/nginx/logs/`
- Laravel logs: `~/laravel-nginx-setup/laravel/logs/`
- Container logs: `docker-compose logs [service-name]`

## Verification Commands

```bash
# Check everything is working
cd ~/laravel-nginx-setup

# Run health check
./health-check.sh

# Test endpoints
curl -v http://localhost
curl -v http://localhost/api/test.php

# Check container networking
docker exec nginx-proxy nslookup laravel-app
```

## Next Steps

1. **Add Database**: Include MySQL/PostgreSQL container
2. **SSL/HTTPS**: Configure SSL certificates
3. **Laravel Features**: Add real Laravel routes and controllers
4. **Monitoring**: Add logging and monitoring tools
5. **Scaling**: Configure load balancing for multiple Laravel instances

This setup provides a solid foundation for a production-ready Laravel application with Nginx reverse proxy!