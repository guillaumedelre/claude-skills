---
name: "frankenphp-1"
description: "FrankenPHP 1.x reference for modern PHP application server built on Caddy. Use when working with FrankenPHP configuration, worker mode, deployment, performance tuning, Docker setup, Symfony integration, environment variables, or any FrankenPHP-related setup. Triggers on: FrankenPHP, worker mode, Caddy, PHP application server, Symfony Runtime, FrankenPHP deployment, Docker FrankenPHP, Caddyfile, Early Hints, HTTP/3, worker script, FRANKENPHP_CONFIG, Symfony FrankenPHP, production deployment, performance optimization."
---

# FrankenPHP 1.x

GitHub: https://github.com/dunglas/frankenphp
Docs: https://frankenphp.dev/

## Overview

FrankenPHP is a modern PHP application server written in Go, built on top of the Caddy web server. It provides worker mode for ultra-fast performance, automatic HTTPS, HTTP/2, HTTP/3 support, and seamless integration with Symfony applications.

## Quick Reference

### Basic Dockerfile

```dockerfile
FROM dunglas/frankenphp

# Install PHP extensions
RUN install-php-extensions \
    pdo_pgsql \
    intl \
    zip \
    apcu \
    opcache

# Copy application
COPY . /app

# Set working directory
WORKDIR /app

# Install dependencies
RUN composer install --no-dev --optimize-autoloader

# Set permissions
RUN chown -R www-data:www-data /app/var

# Expose port
EXPOSE 80 443

# Start FrankenPHP
CMD ["frankenphp", "run", "--config", "/etc/caddy/Caddyfile"]
```

### Caddyfile Configuration

```caddyfile
{
    frankenphp {
        worker ./public/index.php
        num_threads 4
    }
}

localhost {
    root * /app/public
    php_server
}
```

### Symfony Integration

```yaml
# config/packages/framework.yaml
framework:
    http_method_override: false
    handle_all_throwables: true
    php_errors:
        log: true
    runtime: 'Runtime\FrankenPhpSymfony\Runtime'
```

### Worker Mode Script

```php
<?php
// public/index.php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context) {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

### Docker Compose

```yaml
services:
  app:
    build: .
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/app
      - caddy_data:/data
      - caddy_config:/config
    environment:
      SERVER_NAME: ":80"
      FRANKENPHP_CONFIG: "worker ./public/index.php"
      APP_ENV: prod
      APP_DEBUG: 0

volumes:
  caddy_data:
  caddy_config:
```

### Environment Variables

```bash
# Server configuration
SERVER_NAME=:80,:443
CADDY_GLOBAL_OPTIONS=""

# FrankenPHP configuration
FRANKENPHP_CONFIG="worker ./public/index.php; num_threads 4"

# PHP configuration
PHP_INI_SCAN_DIR="/usr/local/etc/php/conf.d:/app/docker/php"

# Application
APP_ENV=prod
APP_DEBUG=0
APP_SECRET=your-secret-key
```

### Production Dockerfile

```dockerfile
FROM dunglas/frankenphp:latest-php8.3 AS frankenphp_base

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN install-php-extensions \
    pdo_pgsql \
    pgsql \
    intl \
    zip \
    apcu \
    opcache \
    redis

# Configure PHP for production
COPY docker/php/prod/php.ini $PHP_INI_DIR/conf.d/app.ini
COPY docker/php/prod/opcache.ini $PHP_INI_DIR/conf.d/opcache.ini

# Copy Caddyfile
COPY docker/frankenphp/Caddyfile /etc/caddy/Caddyfile

FROM frankenphp_base AS app_builder

WORKDIR /app

# Copy composer files
COPY composer.json composer.lock symfony.lock ./

# Install composer dependencies
RUN composer install --no-dev --optimize-autoloader --no-scripts

# Copy application
COPY . .

# Build assets (if needed)
RUN php bin/console asset-map:compile

FROM frankenphp_base

WORKDIR /app

# Copy application from builder
COPY --from=app_builder /app ./

# Set permissions
RUN chown -R www-data:www-data var

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

EXPOSE 80 443

CMD ["frankenphp", "run", "--config", "/etc/caddy/Caddyfile"]
```

### Performance Tuning

```ini
; docker/php/prod/php.ini
memory_limit = 512M
max_execution_time = 30
upload_max_filesize = 10M
post_max_size = 10M

; docker/php/prod/opcache.ini
opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0
opcache.save_comments = 1
opcache.enable_file_override = 1
opcache.preload = /app/config/preload.php
opcache.preload_user = www-data
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: frankenphp
        image: your-registry/app:latest
        ports:
        - containerPort: 80
        - containerPort: 443
        env:
        - name: SERVER_NAME
          value: ":80"
        - name: FRANKENPHP_CONFIG
          value: "worker ./public/index.php; num_threads 4"
        - name: APP_ENV
          value: "prod"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

## Full Documentation

For complete details including advanced Caddyfile configuration, custom build with CGO, Early Hints, HTTP/3 setup, graceful reload, debugging, monitoring, security hardening, and performance benchmarking, see [references/frankenphp.md](references/frankenphp.md).
