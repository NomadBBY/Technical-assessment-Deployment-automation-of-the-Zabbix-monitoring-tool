# Zabbix Docker Compose Stack

A complete Zabbix monitoring solution deployed using Docker Compose with separate server and client networks.

## Components

### Server Services (server-compose.yml)

- **MySQL Server** (`mysql-server`) - MySQL 8.0 database for Zabbix server (172.168.1.10)
- **MySQL Proxy** (`mysql-proxy`) - Dedicated MySQL database for Zabbix proxy (172.168.1.15)
- **Zabbix Server** (`zabbix_server`) - Core monitoring server (172.168.1.20)
- **Zabbix Frontend** (`zabbix_frontend`) - Apache-based web interface (172.168.1.30)
- **Zabbix Proxy** (`zabbix_proxy`) - Distributed monitoring proxy (172.168.1.40 / 172.168.2.10)
- **Zabbix Server Agent** (`zabbix_server_agent`) - Agent for monitoring the server (172.168.1.51)
- **Zabbix Proxy Agent** (`zabbix_proxy_agent`) - Agent for monitoring the proxy (172.168.1.52)

### Client Services (client-compose.yml)

- **Zabbix Agents** (`zabbix_agent_one/two/three`) - Agents for monitored hosts (172.168.2.20-22)
- **Ansible Controller** (`ansible_controller`) - Configuration management (172.168.2.30)

### Networks

- **docker-cloud** (172.168.1.0/24) - Server-side network
- **secure-network** (172.168.2.0/24) - Client-side network
- **Note:** Zabbix Proxy bridges both networks

### Volumes

- **mysql-data** - Persistent storage for Zabbix server database
- **mysql-proxy-data** - Persistent storage for Zabbix proxy database

## Port Mappings

| Service | Host Port | Container Port | Description |
|---------|-----------|----------------|-------------|
| MySQL Server | 3306 | 3306 | Main database access |
| MySQL Proxy | 3307 | 3306 | Proxy database access |
| Zabbix Server | 10051 | 10051 | Server trapper port |
| Zabbix Frontend | 80 | 8080 | Web interface (HTTP) |
| Zabbix Proxy | 10061 | 10051 | Proxy trapper port |
| Server Agent | 10050 | 10050 | Server agent port |
| Proxy Agent | 10052 | 10050 | Proxy agent port |
| Client Agents | varies | 10050 | Client agent ports |
| Ansible | 2222 | 22 | SSH access |

## IP Address Reference

### Docker-Cloud Network (172.168.1.0/24)
```
172.168.1.1   - Gateway
172.168.1.10  - MySQL Server
172.168.1.15  - MySQL Proxy
172.168.1.20  - Zabbix Server
172.168.1.30  - Zabbix Frontend
172.168.1.40  - Zabbix Proxy (server-side)
172.168.1.51  - Zabbix Server Agent
172.168.1.52  - Zabbix Proxy Agent
```

### Secure Network (172.168.2.0/24)
```
172.168.2.1   - Gateway
172.168.2.10  - Zabbix Proxy (client-side)
172.168.2.20  - Zabbix Agent One
172.168.2.21  - Zabbix Agent Two
172.168.2.22  - Zabbix Agent Three
172.168.2.30  - Ansible Controller
```

## Quick Start

### 1. Start Server Services

```bash
docker compose up -d
```

### 2. Start Client Services (Optional)

To run client agents on a separate network:

```bash
docker compose -f client-compose.yml up -d
```

### 3. Access Zabbix Web Interface

Open your browser and navigate to:
- **HTTP:** http://localhost
- **HTTPS:** https://localhost:443

### 4. Default Login Credentials

```
Username: Admin
Password: zabbix
```

**Note:** The username has a capital "A".

### 5. Check Service Status

```bash
docker compose ps
docker compose -f client-compose.yml ps  # For client services
```

### 6. View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f zabbix_server
docker compose logs -f zabbix_frontend
```

## Configuration

### Network Architecture

This setup uses two separate Docker Compose files with different networks:

**Server Network (compose.yml):**
- Subnet: 172.168.1.0/24
- Contains: MySQL, Zabbix Server, Frontend, Proxy

**Client Network (client-compose.yml):**
- Subnet: 172.168.2.0/24
- Contains: Zabbix Agent(s) for monitored hosts

### Database Credentials

Default MySQL credentials (configured in `compose.yml`):

```
Root Password: root_password
Database: zabbix
User: zabbix
Password: zabbix_password
```

### Timezone

The frontend is configured for `America/New_York` timezone. To change it, modify the `PHP_TZ` environment variable in the `zabbix_frontend` service.

### Zabbix Proxy Configuration

The proxy uses a dedicated MySQL database (`mysql-proxy`) on port 3307 with database name `zabbix_proxy`.

## Configuring Monitoring

### Adding the Zabbix Proxy

1. **Log in to Zabbix frontend:** http://localhost (Admin/zabbix)
2. **Navigate to:** Data collection → Proxies
3. **Click:** Create proxy
4. **Configuration:**
   - **Proxy name:** `zabbix-proxy-mysql` (must match exactly)
   - **Proxy mode:** Active
   - **Proxy address:** (leave empty for active mode)
5. **Click:** Add

The proxy will connect within 60 seconds. Verify in Data collection → Proxies.

### Adding Monitored Hosts

#### For Server Agent (172.168.1.51)

1. Go to **Data collection** → **Hosts** → **Create host**
2. **Host configuration:**
   - **Host name:** `Zabbix server`
   - **Templates:** Add "Linux by Zabbix agent"
   - **Host groups:** Zabbix servers
3. **Add Interface:**
   - **Type:** Agent
   - **IP address:** `172.168.1.51`
   - **Port:** `10050`
   - **Connect to:** IP
4. **Click:** Add

#### For Proxy Agent (172.168.1.52)

1. Go to **Data collection** → **Hosts** → **Create host**
2. **Host configuration:**
   - **Host name:** `Zabbix proxy`
   - **Templates:** Add "Linux by Zabbix agent"
   - **Host groups:** Proxies
3. **Add Interface:**
   - **Type:** Agent
   - **IP address:** `172.168.1.52`
   - **Port:** `10050`
   - **Connect to:** IP
4. **Click:** Add

#### For Client Agents

1. Go to **Data collection** → **Hosts** → **Create host**
2. **Host configuration:**
   - **Host name:** Match the `ZBX_HOSTNAME` in client-compose.yml
   - **Templates:** Add "Linux by Zabbix agent"
   - **Monitored by proxy:** Select `zabbix-proxy-mysql`
3. **Add Interface:**
   - **Type:** Agent
   - **IP address:** Use the agent's IP (172.168.2.20-22)
   - **Port:** `10050`
4. **Click:** Add

### Important Notes

- Always use **container port 10050** when configuring agent interfaces (not the host port)
- For hosts on the secure-network, configure them to be monitored by the proxy
- Host names must match the `ZBX_HOSTNAME` environment variable in the compose file
- After adding hosts, reload config: `docker exec zabbix-server zabbix_server -R config_cache_reload`

### Proxy Database

The proxy uses a separate database named `zabbix_proxy` in the same MySQL instance.

## Management Commands

### Stop All Services

```bash
# Stop server services
docker compose -f server-compose.yml down

# Stop client services
docker compose -f client-compose.yml down
```

### Stop and Remove Volumes

```bash
docker compose -f server-compose.yml down -v
```

### Restart Services

```bash
# Restart all server services
docker compose -f server-compose.yml restart

# Restart client services
docker compose -f client-compose.yml restart
```

### Restart Specific Service

```bash
docker compose -f server-compose.yml restart zabbix_server
docker compose -f client-compose.yml restart zabbix_agent_one
```

### Update Images

```bash
# Update server images
docker compose -f server-compose.yml pull
docker compose -f server-compose.yml up -d

# Update client images
docker compose -f client-compose.yml pull
docker compose -f client-compose.yml up -d
```

### Reload Zabbix Configuration

```bash
docker exec zabbix-server zabbix_server -R config_cache_reload
```

### Test Agent Connectivity

```bash
# Test server agent
docker exec zabbix-server zabbix_get -s 172.168.1.51 -k agent.ping

# Test proxy agent
docker exec zabbix-server zabbix_get -s 172.168.1.52 -k agent.ping

# Test client agent
docker exec zabbix-server zabbix_get -s 172.168.2.20 -k agent.ping
```

### Access Ansible Controller

```bash
docker exec -it ansible-controller /bin/sh
```

## Troubleshooting

### Host Status "Unknown" or Greyed Out

If a host appears greyed out or shows "Unknown" status:

1. **Enable the host:**
   - Go to Data collection → Hosts
   - Check if "Status" column shows "Enabled" (green)
   - If greyed out, click on the host and enable it

2. **Verify templates are linked:**
   - Edit the host
   - Make sure appropriate templates are added (e.g., "Linux by Zabbix agent")

3. **Check agent connectivity:**
   ```bash
   docker exec zabbix-server zabbix_get -s <agent_ip> -k agent.ping
   ```

4. **Reload configuration:**
   ```bash
   docker exec zabbix-server zabbix_server -R config_cache_reload
   ```

5. **Wait for data collection:**
   - Initial data collection may take 1-2 minutes
   - Refresh the web interface

### Check Container Logs

```bash
# Server logs
docker logs zabbix-server --tail 50

# Frontend logs
docker logs zabbix-frontend --tail 50

# Proxy logs
docker logs zabbix-proxy --tail 50

# Agent logs
docker logs zabbix-server-agent --tail 20
docker logs zabbix-proxy-agent --tail 20

# Database logs
docker logs mysql-server --tail 50
docker logs mysql-proxy --tail 50
```

### Verify Network Connectivity

```bash
# Inspect server network
docker network inspect server-compose_docker-cloud

# Inspect client network
docker network inspect zabbix-client-compose_secure-network

# Test connectivity between containers
docker exec zabbix-server sh -c 'echo "GET / HTTP/1.0\n\n" | nc 172.168.1.30 8080'
```

### Proxy Not Connecting

If proxy shows "Proxy 'zabbix-proxy-mysql' not found":

1. **Verify proxy is registered in Zabbix:**
   - Go to Data collection → Proxies
   - Proxy name must be exactly `zabbix-proxy-mysql`
   - Proxy mode should be "Active"

2. **Check proxy logs:**
   ```bash
   docker logs zabbix-proxy
   ```

3. **Verify proxy can reach server:**
   ```bash
   docker exec zabbix-proxy sh -c 'echo "test" | nc 172.168.1.20 10051'
   ```

### Database Connection Issues

If you see MySQL connection errors:

1. Ensure MySQL container is running and healthy
2. Check database credentials in environment variables
3. Wait for MySQL to fully initialize (may take 30-60 seconds on first start)

### Reset Everything

```bash
docker compose down -v
docker compose up -d
```

**Warning:** This will delete all monitoring data and configuration.

## Architecture

```
Server Network (172.168.1.0/24)
┌─────────────────────────────────────────┐
│                                         │
│  ┌─────────────────┐                   │
│  │ MySQL Database  │                   │
│  │   (Port 3306)   │                   │
│  └────────┬────────┘                   │
│           │                             │
│           ▼                             │
│  ┌─────────────────┐                   │
│  │  Zabbix Server  │                   │
│  │   (Port 10051)  │                   │
│  └────────┬────────┘                   │
│           │                             │
│      ┌────┴────┐                       │
│      ▼         ▼                       │
│  ┌────────┐ ┌──────┐                  │
│  │Frontend│ │Proxy │                  │
│  │(80/443)│ │(10061│                  │
│  └────────┘ └──────┘                  │
│                                         │
└─────────────────────────────────────────┘

Client Network (172.168.2.0/24)
┌─────────────────────────────────────────┐
│                                         │
│  ┌─────────────────┐                   │
│  │  Zabbix Agent   │                   │
│  │   (Port 10050)  │                   │
│  └─────────────────┘                   │
│                                         │
│  (Connects to Server at 172.168.1.x)   │
│                                         │
└─────────────────────────────────────────┘
```

## Security Notes

⚠️ **Important:** This configuration uses default passwords suitable for development/testing only.

For production use:
- Change all default passwords
- Use strong, unique passwords
- Configure SSL/TLS certificates
- Restrict network access using firewalls
- Regularly update images for security patches

## Additional Resources

- [Zabbix Documentation](https://www.zabbix.com/documentation/current)
- [Zabbix Docker Images](https://hub.docker.com/u/zabbix)
- [Zabbix Community Forums](https://www.zabbix.com/forum/)
