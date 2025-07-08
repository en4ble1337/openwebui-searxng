# Complete Guide: SearXNG with OpenWebUI Integration

This comprehensive guide provides step-by-step instructions for setting up SearXNG in an LXC container with Docker Compose and integrating it with OpenWebUI. The configuration includes specific search engines and JSON API support for optimal integration.

## Table of Contents
1. [Prerequisites and LXC Container Setup](#1-prerequisites-and-lxc-container-setup)
2. [Docker Installation on LXC](#2-docker-installation-on-lxc)
3. [SearXNG Installation and Configuration](#3-searxng-installation-and-configuration)
4. [OpenWebUI Integration](#4-openwebui-integration)
5. [Testing and Validation](#5-testing-and-validation)
6. [Service Management](#6-service-management)
7. [Troubleshooting](#7-troubleshooting)

## 1. Prerequisites and LXC Container Setup

### System Requirements
- **Proxmox VE environment**
- **Resources**: 4 vCPU, 8GB RAM, 120GB storage minimum
- **Network**: Static IP address (example: 10.1.20.139)
- **OS**: Ubuntu 22.04 LTS

### Create LXC Container

**Step 1: Create container via Proxmox web interface**
- Template: Ubuntu 22.04 LTS
- Container ID: Choose available ID
- Hostname: `searxng-10` (or preferred name)
- Resources: 4 vCPU, 8GB RAM, 120GB storage
- Network: Bridge with static IP (10.1.20.139)

**Step 2: Enable Docker support**

On the Proxmox host, enable nesting:
```bash
# Enable nesting for Docker compatibility
pct set  -features nesting=1,keyctl=1

# Start the container
pct start 

# Enter the container
pct enter 
```

## 2. Docker Installation on LXC

### System Preparation

**Step 1: Update system and install dependencies**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release git
```

### Install Docker

**Step 2: Add Docker repository**
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Step 3: Install Docker and Docker Compose**
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Configure Docker for LXC

**Step 4: Create Docker daemon configuration**
```bash
sudo nano /etc/docker/daemon.json
```

Add the following content:
```json
{
  "storage-driver": "vfs",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Step 5: Enable and start Docker**
```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ${USER}
```

**Important**: Log out and back in to apply group changes.

**Step 6: Verify installation**
```bash
docker --version
docker compose version
docker run hello-world
```

## 3. SearXNG Installation and Configuration

### Clone Repository

**Step 1: Clone SearXNG Docker repository**
```bash
cd /usr/local
sudo git clone https://github.com/searxng/searxng-docker.git
cd searxng-docker
```

### Environment Configuration

**Step 2: Create .env file**
```bash
sudo nano .env
```

Add the following content:
```bash
SEARXNG_BASE_URL=http://10.1.20.139:8080/
SEARXNG_HOSTNAME=10.1.20.139:8080
SEARXNG_UWSGI_WORKERS=4
SEARXNG_UWSGI_THREADS=4
```

### Docker Compose Configuration

**Step 3: Create docker-compose.yaml**
```bash
sudo nano docker-compose.yaml
```

Add the complete configuration:
```yaml
version: "3.7"

services:
  redis:
    container_name: redis
    image: docker.io/valkey/valkey:8-alpine
    command: valkey-server --save 30 1 --loglevel warning
    restart: unless-stopped
    networks:
      - searxng
    volumes:
      - valkey-data2:/data
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"

  searxng:
    container_name: searxng
    image: docker.io/searxng/searxng:latest
    restart: unless-stopped
    networks:
      - searxng
    ports:
      - "8080:8080"
    volumes:
      - ./searxng:/etc/searxng:rw
      - searxng-data:/var/cache/searxng:rw
    env_file:
      - .env
    environment:
      - SEARXNG_BASE_URL=${SEARXNG_BASE_URL}
      - UWSGI_WORKERS=${SEARXNG_UWSGI_WORKERS:-4}
      - UWSGI_THREADS=${SEARXNG_UWSGI_THREADS:-4}
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"

networks:
  searxng:

volumes:
  valkey-data2:
  searxng-data:
```

### SearXNG Settings Configuration

**Step 4: Create searxng directory and settings**
```bash
sudo mkdir -p searxng
sudo nano searxng/settings.yml
```

Add the complete settings configuration:
```yaml
# see https://docs.searxng.org/admin/settings/settings.html#settings-use-default-settings
use_default_settings: true
server:
  bind_address: "0.0.0.0:8080"
  # base_url is defined in the SEARXNG_BASE_URL environment variable, see .env and docker-compose.yml
  secret_key: "4513e306ea5b5136b2bfa279cbc89c5061bf15d0e8bb57916766abed3e4fc0b6"  # change this!
  limiter: false  # enable this when running the instance for a public usage on the internet
  image_proxy: true
ui:
  static_use_hash: true
redis:
  url: redis://redis:6379/0
search:
  formats:
    - html
    - json
```

**Step 5: Generate unique secret key**
```bash
# Generate a new secret key
openssl rand -hex 32

# Replace the secret_key in settings.yml with the generated key
sudo sed -i "s|4513e306ea5b5136b2bfa279cbc89c5061bf15d0e8bb57916766abed3e4fc0b6|$(openssl rand -hex 32)|g" searxng/settings.yml
```

### Start SearXNG

**Step 6: Launch the services**
```bash
sudo docker compose up -d
```

**Step 7: Verify containers are running**
```bash
sudo docker ps
sudo docker compose logs searxng
```

Look for these success indicators:
- `WSGI app 0 (mountpoint='') ready in X seconds`
- Port binding shows `0.0.0.0:8080->8080/tcp`

## 4. OpenWebUI Integration

### Configuration Methods

**Method 1: Environment Variables (Recommended)**

Add to your OpenWebUI docker-compose.yaml:
```yaml
services:
  open-webui:
    environment:
      ENABLE_RAG_WEB_SEARCH: "true"
      RAG_WEB_SEARCH_ENGINE: "searxng"
      RAG_WEB_SEARCH_RESULT_COUNT: 5
      RAG_WEB_SEARCH_CONCURRENT_REQUESTS: 10
      SEARXNG_QUERY_URL: "http://10.1.20.139:8080/search?q=&format=json&engines=duckduckgo,google,brave,bing"
```

**Method 2: GUI Configuration**

1. Access OpenWebUI admin panel
2. Navigate to **Admin Panel** → **Settings** → **Web Search**
3. Configure the following settings:
   - **Enable Web Search**: Toggle ON
   - **Web Search Engine**: Select `searxng`
   - **Searxng Query URL**: `http://10.1.20.139:8080/search?q=&format=json&engines=duckduckgo,google,brave,bing`
   - **Search Result Count**: 5
   - **Concurrent Requests**: 10
4. Save changes

### Key Integration Points

**Critical URL Format**: The search query URL must include:
- `format=json` for JSON response
- `engines=duckduckgo,google,brave,bing` for specific search engines
- `` placeholder for dynamic query replacement

**Complete URL Template**:
```
http://10.1.20.139:8080/search?q=&format=json&engines=duckduckgo,google,brave,bing
```

## 5. Testing and Validation

### Direct API Testing

**Step 1: Test SearXNG API directly**
```bash
# Test basic connectivity
curl http://10.1.20.139:8080/

# Test JSON API with specific engines
curl "http://10.1.20.139:8080/search?q=test&format=json&engines=duckduckgo,google,brave,bing"
```

**Step 2: Test from OpenWebUI container**
```bash
docker exec -it open-webui curl "http://10.1.20.139:8080/search?q=test&format=json&engines=duckduckgo,google,brave,bing"
```

### OpenWebUI Integration Testing

**Step 3: Test in OpenWebUI interface**
1. Open a chat in OpenWebUI
2. Click the **+** button next to the message input
3. Toggle **Web Search** ON
4. Ask a question requiring web search (e.g., "What are the latest news about AI?")
5. Verify search results are included in the response

### Validation Checklist

- [ ] SearXNG containers are running (`docker ps`)
- [ ] Port 8080 is accessible externally
- [ ] JSON API returns valid responses
- [ ] OpenWebUI can connect to SearXNG
- [ ] Web search toggle works in OpenWebUI
- [ ] Search results are integrated into chat responses

## 6. Service Management

### Daily Operations

**Start services**:
```bash
sudo docker compose up -d
```

**Stop services**:
```bash
sudo docker compose down
```

**Restart services**:
```bash
sudo docker compose restart
```

**View logs**:
```bash
# View all logs
sudo docker compose logs -f

# View specific service logs
sudo docker compose logs -f searxng
sudo docker compose logs -f redis
```

### Monitoring

**Check container status**:
```bash
sudo docker ps
sudo docker stats
```

**Monitor resource usage**:
```bash
htop
df -h
free -h
```

**Check network connectivity**:
```bash
netstat -tlnp | grep 8080
ss -tlnp | grep 8080
```

### Updates and Maintenance

**Update SearXNG**:
```bash
sudo docker compose pull
sudo docker compose up -d
```

**Clean up Docker resources**:
```bash
sudo docker system prune -f
sudo docker volume prune -f
```

**Backup configuration**:
```bash
sudo tar -czf searxng-backup-$(date +%Y%m%d).tar.gz searxng/ .env docker-compose.yaml
```

## 7. Troubleshooting

### Common Issues and Solutions

#### Connection Refused Errors
**Problem**: `Failed to connect to 10.1.20.139 port 8080`

**Solutions**:
1. Check port binding: `docker ps` should show `0.0.0.0:8080->8080/tcp`
2. Verify firewall settings
3. Ensure `bind_address: "0.0.0.0:8080"` in settings.yml

#### YAML Syntax Errors
**Problem**: `expected , but found ''`

**Solution**: Verify proper YAML indentation (2 spaces) and structure:
```yaml
server:
  bind_address: "0.0.0.0:8080"  # Correct
# NOT: server: "0.0.0.0:8080"  # Wrong
```

#### Container Won't Start
**Problem**: `unable to load app 0 (mountpoint='') (callable not found or import error)`

**Solutions**:
1. Check settings.yml syntax
2. Verify secret key is properly set
3. Ensure proper file permissions: `sudo chown -R root:root searxng/`

#### JSON API Issues
**Problem**: OpenWebUI can't parse search results

**Solutions**:
1. Verify `format=json` in the query URL
2. Ensure `json` format is enabled in settings.yml:
   ```yaml
   search:
     formats:
       - html
       - json
   ```

#### Search Engine Errors
**Problem**: Specific engines not working

**Solutions**:
1. Test individual engines: `engines=duckduckgo` or `engines=google`
2. Check SearXNG logs for engine-specific errors
3. Verify engine availability in SearXNG settings

### Debug Commands

```bash
# Check container logs
sudo docker compose logs searxng | tail -50

# Test API endpoints
curl -v "http://10.1.20.139:8080/search?q=test&format=json"

# Check network connectivity
ping 10.1.20.139
telnet 10.1.20.139 8080

# Verify Docker network
sudo docker network ls
sudo docker network inspect searxng-docker_searxng
```

### Performance Optimization

**Adjust worker settings** in .env:
```bash
SEARXNG_UWSGI_WORKERS=8  # Increase for high traffic
SEARXNG_UWSGI_THREADS=6  # Adjust based on CPU cores
```

**Monitor resource usage**:
```bash
sudo docker stats searxng redis
```

## Summary

This guide provides a complete setup for SearXNG with OpenWebUI integration featuring:

- **Dedicated LXC container** for isolation and resource management
- **Docker Compose deployment** for easy management
- **Multi-engine search** with DuckDuckGo, Google, Brave, and Bing
- **JSON API support** for seamless OpenWebUI integration
- **Comprehensive monitoring** and troubleshooting procedures

The key success factors are:
1. Proper LXC configuration with Docker nesting
2. Correct port binding (`0.0.0.0:8080`)
3. Valid YAML configuration with JSON format support
4. Specific search engine configuration
5. Proper OpenWebUI integration URL format

Following this guide will result in a fully functional SearXNG instance that enhances OpenWebUI with powerful web search capabilities across multiple search engines.
