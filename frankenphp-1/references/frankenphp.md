# FrankenPHP 1.x Complete Reference

## Table of Contents

1. [Installation & Setup](#installation--setup)
2. [Worker Mode](#worker-mode)
3. [Caddyfile Configuration](#caddyfile-configuration)
4. [Docker Setup](#docker-setup)
5. [Symfony Integration](#symfony-integration)
6. [Environment Variables](#environment-variables)
7. [Performance Tuning](#performance-tuning)
8. [Production Deployment](#production-deployment)
9. [Kubernetes](#kubernetes)
10. [Monitoring & Debugging](#monitoring--debugging)
11. [Security](#security)
12. [Advanced Features](#advanced-features)
13. [Troubleshooting](#troubleshooting)

---

## Installation & Setup

### Standalone Binary

```bash
# Download latest version
curl https://frankenphp.dev/install.sh | sh

# Run
frankenphp run
```

### Docker

```bash
docker run -v $PWD:/app -p 80:80 -p 443:443 dunglas/frankenphp
```

### Composer (Symfony)

```bash
composer require runtime/frankenphp-symfony
```

---

## Worker Mode

Worker mode keeps the application in memory, dramatically improving performance by eliminating bootstrap overhead.

### Basic Worker Script

```php
<?php
// public/index.php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context) {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

### Enable Worker Mode

```caddyfile
{
    frankenphp {
        worker ./public/index.php
        num_threads 4
    }
}
```

### Worker Lifecycle

```php
// Custom worker script with lifecycle hooks
<?php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

// Called once when worker starts
$kernel = new Kernel($_SERVER['APP_ENV'] ?? 'prod', (bool) ($_SERVER['APP_DEBUG'] ?? false));
$kernel->boot();

// Called for each request
$handler = static function () use ($kernel): void {
    // Clear Doctrine entity manager between requests
    $kernel->getContainer()->get('doctrine')->getManager()->clear();

    // Clear request-scoped services
    $kernel->getContainer()->get('services_resetter')->reset();
};

return $handler;
```

### Worker Best Practices

1. **Clear stateful services** between requests (Doctrine EM, session, etc.)
2. **Avoid global state** - Worker persists across requests
3. **Memory management** - Watch for memory leaks
4. **Connection pooling** - Reuse database/cache connections
5. **Error handling** - Worker should never crash

### Restart Worker on Changes

```caddyfile
{
    frankenphp {
        worker ./public/index.php
        watch /app/config
        watch /app/src
    }
}
```

---

## Caddyfile Configuration

### Basic Configuration

```caddyfile
{
    # Global options
    debug
    admin off

    frankenphp {
        worker ./public/index.php
        num_threads {$NUM_THREADS:4}
    }
}

:80 {
    root * /app/public

    # PHP-FPM-like behavior
    php_server

    # Enable compression
    encode gzip

    # Access log
    log {
        output file /var/log/access.log
        format json
    }
}
```

### Advanced Configuration

```caddyfile
{
    frankenphp {
        worker ./public/index.php
        num_threads 8

        # Custom directives
        max_request_body_size 10MB
    }
}

https://example.com {
    root * /app/public

    # Custom header
    header {
        X-Powered-By "FrankenPHP"
        -Server
    }

    # PHP server with custom config
    php_server {
        resolve_root_symlink
    }

    # Compression
    encode zstd gzip

    # Static file caching
    @static {
        path *.css *.js *.ico *.png *.jpg *.jpeg *.gif *.svg *.woff *.woff2
    }
    header @static Cache-Control "public, max-age=31536000, immutable"

    # API routes
    @api {
        path /api/*
    }
    header @api {
        Access-Control-Allow-Origin "*"
        Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
    }

    # Error pages
    handle_errors {
        respond "{err.status_code} {err.status_text}"
    }

    # Logging
    log {
        output file /var/log/access.log {
            roll_size 100MB
            roll_keep 10
        }
        format json
        level info
    }
}
```

### Environment-Specific Caddyfile

```caddyfile
# Production
{
    frankenphp {
        worker ./public/index.php
        num_threads 16
    }
}

https://app.example.com {
    root * /app/public
    php_server

    encode zstd gzip

    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        X-XSS-Protection "1; mode=block"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    # Rate limiting (if using caddy-rate-limit plugin)
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 100
            window 1m
        }
    }

    # Health check endpoint
    @health {
        path /health
    }
    respond @health 200

    log {
        output file /var/log/app.log
        format json
        level warn
    }
}
```

---

## Docker Setup

### Multi-Stage Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.4

# Base image with PHP and extensions
FROM dunglas/frankenphp:latest-php8.3 AS base

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
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
    redis \
    amqp

# Configure PHP
COPY docker/php/php.ini $PHP_INI_DIR/conf.d/app.ini
COPY docker/php/opcache.ini $PHP_INI_DIR/conf.d/opcache.ini

# Copy Caddyfile
COPY docker/frankenphp/Caddyfile /etc/caddy/Caddyfile

WORKDIR /app

# Builder stage
FROM base AS builder

# Copy composer files
COPY composer.json composer.lock symfony.lock ./
COPY --link --from=composer:latest /usr/bin/composer /usr/bin/composer

# Install dependencies
ENV COMPOSER_ALLOW_SUPERUSER=1
RUN composer install --no-dev --optimize-autoloader --no-scripts --no-progress

# Copy application
COPY . .

# Build assets
RUN composer dump-autoload --optimize --classmap-authoritative && \
    php bin/console asset-map:compile && \
    php bin/console cache:warmup

# Production stage
FROM base

WORKDIR /app

# Copy application from builder
COPY --chown=www-data:www-data --from=builder /app ./

# Set permissions
RUN chown -R www-data:www-data var

# Health check
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

USER www-data

EXPOSE 80 443 443/udp

ENTRYPOINT ["frankenphp", "run", "--config", "/etc/caddy/Caddyfile"]
```

### Docker Compose (Development)

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      target: base
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"  # HTTP/3
    volumes:
      - ./:/app
      - caddy_data:/data
      - caddy_config:/config
    environment:
      SERVER_NAME: ":80"
      FRANKENPHP_CONFIG: "worker ./public/index.php; num_threads 4"
      APP_ENV: dev
      APP_DEBUG: 1
      DATABASE_URL: postgresql://app:!ChangeMe!@database:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      - database
      - redis

  database:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: "!ChangeMe!"
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  caddy_data:
  caddy_config:
  db_data:
```

### Docker Compose (Production)

```yaml
version: '3.8'

services:
  app:
    image: ${REGISTRY}/app:${TAG:-latest}
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    environment:
      SERVER_NAME: "example.com:443"
      FRANKENPHP_CONFIG: "worker ./public/index.php; num_threads 16"
      APP_ENV: prod
      APP_DEBUG: 0
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s

volumes:
  caddy_data:
  caddy_config:
```

---

## Symfony Integration

### Install Runtime

```bash
composer require runtime/frankenphp-symfony
```

### Framework Configuration

```yaml
# config/packages/framework.yaml
framework:
    http_method_override: false
    handle_all_throwables: true
    php_errors:
        log: true
    runtime: 'Runtime\FrankenPhpSymfony\Runtime'

    # HTTP cache configuration for worker mode
    http_cache:
        enabled: true
```

### Clear Services Between Requests

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    # Reset Doctrine between requests in worker mode
    App\EventSubscriber\WorkerSubscriber:
        tags:
            - { name: kernel.event_subscriber }
```

```php
// src/EventSubscriber/WorkerSubscriber.php
namespace App\EventSubscriber;

use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class WorkerSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private EntityManagerInterface $em
    ) {}

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::TERMINATE => 'onKernelTerminate',
        ];
    }

    public function onKernelTerminate(TerminateEvent $event): void
    {
        // Clear Doctrine EM to prevent memory leaks
        $this->em->clear();

        // Close connections if needed
        if (!$this->em->getConnection()->ping()) {
            $this->em->getConnection()->close();
            $this->em->getConnection()->connect();
        }
    }
}
```

---

## Environment Variables

### Server Configuration

```bash
# Server name (domains or :port)
SERVER_NAME=":80,:443"
SERVER_NAME="example.com:443,www.example.com:443"

# Caddy global options
CADDY_GLOBAL_OPTIONS="debug"
```

### FrankenPHP Configuration

```bash
# Worker configuration
FRANKENPHP_CONFIG="worker ./public/index.php; num_threads 8"

# Without worker mode
FRANKENPHP_CONFIG=""
```

### PHP Configuration

```bash
# PHP INI directories
PHP_INI_SCAN_DIR="/usr/local/etc/php/conf.d:/app/docker/php"

# Memory limit
PHP_MEMORY_LIMIT=512M

# Max execution time
PHP_MAX_EXECUTION_TIME=30
```

### Application Configuration

```bash
APP_ENV=prod
APP_DEBUG=0
APP_SECRET=your-secret-key-here

DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://redis:6379

MESSENGER_TRANSPORT_DSN=amqp://guest:guest@rabbitmq:5672/%2f/messages
```

---

## Performance Tuning

### PHP Configuration

```ini
; docker/php/php.ini
[PHP]
memory_limit = 512M
max_execution_time = 30
max_input_time = 60
upload_max_filesize = 20M
post_max_size = 20M
max_input_vars = 1000

; Realpath cache
realpath_cache_size = 4096K
realpath_cache_ttl = 600

[Date]
date.timezone = UTC
```

### OPcache Configuration

```ini
; docker/php/opcache.ini
[opcache]
opcache.enable = 1
opcache.enable_cli = 0

; Memory
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000

; Performance
opcache.validate_timestamps = 0
opcache.revalidate_freq = 0
opcache.save_comments = 1
opcache.enable_file_override = 1

; JIT
opcache.jit = tracing
opcache.jit_buffer_size = 100M

; Preloading
opcache.preload = /app/config/preload.php
opcache.preload_user = www-data
```

### Preload Script

```php
// config/preload.php
<?php

if (file_exists(__DIR__ . '/../var/cache/prod/App_KernelProdContainer.preload.php')) {
    require __DIR__ . '/../var/cache/prod/App_KernelProdContainer.preload.php';
}
```

### APCu Configuration

```ini
; docker/php/apcu.ini
[apcu]
apc.enabled = 1
apc.shm_size = 128M
apc.ttl = 0
apc.gc_ttl = 3600
apc.enable_cli = 1
```

### Worker Threads

```caddyfile
{
    frankenphp {
        num_threads {$NUM_THREADS:16}
    }
}
```

Rule of thumb: `num_threads = (CPU cores * 2) to (CPU cores * 4)`

---

## Production Deployment

### Build & Push Docker Image

```bash
# Build
docker build -t registry.example.com/app:latest .

# Tag
docker tag registry.example.com/app:latest registry.example.com/app:v1.0.0

# Push
docker push registry.example.com/app:latest
docker push registry.example.com/app:v1.0.0
```

### Deploy with Docker Compose

```bash
# Pull latest image
docker-compose pull

# Restart services
docker-compose up -d --no-deps --build app

# Check logs
docker-compose logs -f app
```

### Zero-Downtime Deployment

```bash
# 1. Pull new image
docker pull registry.example.com/app:v1.0.1

# 2. Start new containers
docker-compose up -d --scale app=6 --no-recreate

# 3. Wait for health checks
sleep 30

# 4. Stop old containers
docker-compose up -d --scale app=3 --no-recreate

# 5. Remove old containers
docker-compose rm -f
```

---

## Kubernetes

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
        image: registry.example.com/app:latest
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: http3
          containerPort: 443
          protocol: UDP
        env:
        - name: SERVER_NAME
          value: ":80"
        - name: FRANKENPHP_CONFIG
          value: "worker ./public/index.php; num_threads 16"
        - name: APP_ENV
          value: "prod"
        - name: APP_DEBUG
          value: "0"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        volumeMounts:
        - name: caddy-data
          mountPath: /data
        - name: caddy-config
          mountPath: /config
      volumes:
      - name: caddy-data
        persistentVolumeClaim:
          claimName: caddy-data
      - name: caddy-config
        persistentVolumeClaim:
          claimName: caddy-config
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  - name: http3
    port: 443
    targetPort: 443
    protocol: UDP
  selector:
    app: app
```

### Ingress (with cert-manager)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: app-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app
            port:
              number: 80
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Monitoring & Debugging

### Access Logs

```caddyfile
{
    log {
        output file /var/log/access.log {
            roll_size 100MB
            roll_keep 10
        }
        format json
        level info
    }
}
```

### Prometheus Metrics

```caddyfile
{
    servers {
        metrics
    }
}

:2019 {
    metrics /metrics
}
```

### Health Check Endpoint

```php
// src/Controller/HealthController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Annotation\Route;

class HealthController
{
    #[Route('/health', methods: ['GET'])]
    public function check(): JsonResponse
    {
        return new JsonResponse(['status' => 'healthy']);
    }
}
```

### Debug Mode

```bash
# Enable debug logging
CADDY_GLOBAL_OPTIONS="debug"

# Check worker status
docker exec -it app_container curl http://localhost:2019/config/
```

---

## Security

### HTTPS Auto-Configuration

FrankenPHP (via Caddy) automatically provisions TLS certificates with Let's Encrypt:

```caddyfile
https://example.com {
    # Automatic HTTPS!
    root * /app/public
    php_server
}
```

### Security Headers

```caddyfile
header {
    # HSTS
    Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

    # Prevent MIME sniffing
    X-Content-Type-Options "nosniff"

    # Clickjacking protection
    X-Frame-Options "SAMEORIGIN"

    # XSS protection
    X-XSS-Protection "1; mode=block"

    # Referrer policy
    Referrer-Policy "strict-origin-when-cross-origin"

    # Content Security Policy
    Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"

    # Remove server header
    -Server
}
```

---

## Advanced Features

### Early Hints (HTTP 103)

```caddyfile
{
    frankenphp {
        early_hints
    }
}
```

### HTTP/3

HTTP/3 is enabled by default in FrankenPHP:

```yaml
# docker-compose.yaml
ports:
  - "443:443"
  - "443:443/udp"  # HTTP/3 QUIC
```

### Custom Build with Extensions

```dockerfile
FROM dunglas/frankenphp:latest-builder AS builder

# Install custom PHP extensions
RUN install-php-extensions imagick

FROM dunglas/frankenphp:latest

COPY --from=builder /usr/local/lib/php/extensions /usr/local/lib/php/extensions
COPY --from=builder /usr/local/etc/php/conf.d /usr/local/etc/php/conf.d
```

---

## Troubleshooting

### Worker Not Starting

Check logs:
```bash
docker logs app_container
```

Common issues:
- Syntax error in worker script
- Missing dependencies
- Permission issues

### Memory Leaks

Monitor memory usage:
```bash
docker stats app_container
```

Solutions:
- Clear Doctrine EM between requests
- Reset request-scoped services
- Check for circular references

### Performance Issues

1. Enable OPcache
2. Use worker mode
3. Increase `num_threads`
4. Enable APCu
5. Use persistent connections
6. Profile with Blackfire/Xdebug

---

## Best Practices

1. **Always use worker mode in production** for maximum performance
2. **Clear stateful services** between requests (Doctrine EM, etc.)
3. **Use persistent connections** for databases and cache
4. **Enable OPcache** and configure properly
5. **Monitor memory** usage in worker mode
6. **Use health checks** for readiness/liveness probes
7. **Configure proper logging** for debugging
8. **Set resource limits** in production
9. **Use HTTP/3** for better performance
10. **Implement graceful shutdown** for zero-downtime deployments
