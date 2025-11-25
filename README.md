# Zabbix Docker Compose Stack

A complete Zabbix monitoring solution deployed using Docker Compose with separate server and client networks automated with Ansible.

## Components

### Server Services (server-compose.yml)

- **MySQL Server** (`mysql-server`) - MySQL 8.0 database for Zabbix server (172.168.1.11)
- **Zabbix Server** (`zabbix_server`) - Main monitoring server (172.168.1.10)
- **Zabbix Frontend** (`zabbix_frontend`) - Apache-based web interface (172.168.1.12)
- **Zabbix Proxy** (`zabbix_proxy`) - Monitoring proxy with SQLite3 (172.168.1.20 / 172.168.2.10)
- **Zabbix Server Agent** (`zabbix_server_agent`) - Agent for monitoring the Zabbix Server (172.168.1.31)
- **Zabbix Proxy Agent** (`zabbix_proxy_agent`) - Agent for monitoring the Zabbix Proxy (172.168.1.32)

### Client Services (client-compose.yml)

- **SNMP Simulator One** (`snmp-sim1`) - SNMP device simulator for Simulated Router #1 (172.168.2.21)
- **SNMP Simulator Two** (`snmp-sim2`) - SNMP device simulator for Simulated Switch #2 (172.168.2.22)
- **SNMP Simulator Three** (`snmp-sim3`) - SNMP device simulator for Simulated Firewall #3 (172.168.2.23)
- **Zabbix Agent 1** (`zabbix-agent1`) - Zabbix agent for network discovery testing (172.168.2.31)
- **Zabbix Agent 2** (`zabbix-agent2`) - Zabbix agent for network discovery testing (172.168.2.32)

### Networks

- **docker-cloud** (172.168.1.0/24) - Server-side network (In requirments "Docker/Cloud")
- **secure-network** (172.168.2.0/24) - Client-side network (In requirments "Secure Network")
- **Note:** Zabbix Proxy bridges both networks

### Volumes

- **mysql-data** - Persistent storage for Zabbix server database
- **zabbix-proxy-data** - Persistent storage for Zabbix proxy SQLite database

## Port Mappings

| Service | Host Port | Container Port | Description |
|---------|-----------|----------------|-------------|
| MySQL Server | 3306 | 3306 | Main database access |
| Zabbix Server | 10051 | 10051 | Server trapper port |
| Zabbix Frontend | 80 | 8080 | Web interface (HTTP) |
| Zabbix Proxy | 10061 | 10051 | Proxy trapper port |
| Server Agent | 10050 | 10050 | Server agent port |
| Proxy Agent | 10052 | 10050 | Proxy agent port |
| SNMP Sim 1 | 1161 | 161/udp | SNMP device simulator 1 |
| SNMP Sim 2 | 1162 | 161/udp | SNMP device simulator 2 |
| SNMP Sim 3 | 1163 | 161/udp | SNMP device simulator 3 |
| Zabbix Agent 1 | 10070 | 10050 | Test agent 1 |
| Zabbix Agent 2 | 10071 | 10050 | Test agent 2 |

## IP Address Reference

### Docker-Cloud Network (172.168.1.0/24)
```
172.168.1.1   - Gateway
172.168.1.10  - Zabbix Server
172.168.1.11  - MySQL Server
172.168.1.12  - Zabbix Frontend
172.168.1.20  - Zabbix Proxy (server-side)
172.168.1.31  - Zabbix Server Agent
172.168.1.32  - Zabbix Proxy Agent
```

### Secure Network (172.168.2.0/24)
```
172.168.2.1   - Gateway
172.168.2.10  - Zabbix Proxy (client-side)
172.168.2.21  - SNMP Simulator 1
172.168.2.22  - SNMP Simulator 2
172.168.2.23  - SNMP Simulator 3
172.168.2.31  - Zabbix Agent 1
172.168.2.32  - Zabbix Agent 2
```

## Quick Start
sleep 20 && \

### 1. Deploy Everything with Ansible

The deployment is now fully automated using Ansible. You can control the cleanup and deployment behavior using the `cleanup_zabbix` variable, which is defined globally in the playbook or can be set in `group_vars/all.yml`.

#### To deploy Zabbix monitoring:
```bash
cd /ansible
ansible-playbook site.yml
```

#### To clean up Zabbix (remove containers, volumes, networks):
Edit `site.yml` or `group_vars/all.yml` and set:
```yaml
cleanup_zabbix: true
```
Then run:
```bash
ansible-playbook site.yml
```

#### To deploy (start everything):
Set:
```yaml
cleanup_zabbix: false
```
Then run:
```bash
ansible-playbook site.yml
```

You can also set variables in `group_vars/all.yml` for global control across all plays and roles.

---

### 2. Access Zabbix Web Interface

Open browser and navigate to:
- **HTTP:** http://localhost

### 3. Default Login Credentials

```
Username: Admin
Password: zabbix
```

**Note:** The username has a capital "A".

## Configuration

### Network Architecture

This setup uses two separate Docker Compose files with two different networks:

**Server Network (server-compose.yml):**
- Subnet: 172.168.1.0/24
- Contains: MySQL, Zabbix Server, Zabbix Frontend, Zabbix Proxy, Server Agent, Proxy Agent

**Client Network (client-compose.yml):**
- Subnet: 172.168.2.0/24
- Contains: SNMP device simulators for testing SNMP monitoring (As requested per task) and two Zabbix Agents that are used for Network Discovery proof of concept

### Zabbix Proxy Configuration

The proxy uses SQLite3 for its database storage, configured with active mode to connect to the Zabbix server at 172.168.1.10.

### SNMP Simulator Configuration

Three SNMP simulators are configured to test SNMP monitoring:
- **SNMPv3 User:** `zabbixUser` with `noAuthNoPriv` security level
- **Data Files:** Each simulator uses a separate `.snmprec` file in the `client/data/` directory

### Network Discovery

The Ansible role zabbix-automation configures network discovery:

**Discovery Rule: "Secure Network"**
- **IP Range:** 172.168.2.30-33
- **Monitored by:** zabbix-snmp-proxy
- **Checks:** Zabbix agent discovery
- **Frequency:** Every 10 seconds
- **Purpose:** Automatically discovers Zabbix agents on the secure network

### Automated Actions and Alerts

The role configures several automated trigger-based actions and discovery actions:

#### Trigger-Based Actions

**1. SNMP Hosts Not Active**
- **Event Source:** Trigger
- **Conditions:**
  - Trigger severity >= Warning
  - Trigger name contains "Linux: No SNMP data collection"
  - Trigger name contains "Linux: Unavailable by ICMP ping"
- **Operations:**
  - Sends email notification to Admin user
  - Subject: "One/All of SNMP hosts not active"
- **Purpose:** Alerts when SNMP devices experience data collection issues or become unreachable

**2. Zabbix Agent Hosts Not Active**
- **Event Source:** Trigger
- **Conditions (any of the following):**
  - High CPU utilization
  - High memory utilization
  - High swap space usage
  - Lack of available memory
  - Load average is too high
  - Zabbix agent is not available
- **Operations:**
  - Sends email notification to Admin user
  - Subject: "One/All of Zabbix Agent hosts not active"
- **Purpose:** Alerts on critical system resource issues or agent unavailability

#### Discovery-Based Actions

**3. Auto Discovery for Zabbix Agents**
- **Event Source:** Network discovery
- **Conditions:** 
  - Discovery rule: "Secure Network"
  - Proxy: "zabbix-snmp-proxy"
  - Service type: Zabbix agent
- **Operations:**
  - Automatically adds discovered hosts
  - Assigns to "Discovered hosts" group
  - Links "Linux by Zabbix agent" template
- **Purpose:** Automatically registers newly discovered Zabbix agents

## Overview Of The zabbix-automation Role

### Global Variable Usage

You can set `cleanup_zabbix` in:
- The top of `ansible/site.yml` (recommended for one-off runs)
- `ansible/group_vars/all.yml` (recommended for persistent configuration)

Example `group_vars/all.yml`:
```yaml
cleanup_zabbix: false  # Set to true to clean up, false to deploy
```

### Structure 
```bash
.
├── ansible.cfg
├── README.md
├── site.yml
└── zabbix-automation
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── inventory
    │   ├── docker_containers.docker.yml
    │   └── docker_inventory.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   ├── add_actions.yml
    │   ├── add_alerts.yml
    │   ├── add_snmp_hosts.yml
    │   ├── add_zabbix_agents.yml
    │   ├── add_zabbix_proxy.yml
    │   ├── enable_discovery.yml
    │   ├── main.yml
    │   └── set_credentials.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml
```

The Ansible role automates the deployment and configuration of Zabbix monitoring components. It includes tasks for:

* Adding SNMP hosts
* Adding Zabbix agents
* Creating Zabbix proxies
* Creating action triggers and alerts
* Enabling network discovery
* Setting up credentials
  
## Architecture

![Architecture Diagram](architecture.png)

## Security Notes

⚠️ **Important:** This configuration uses default passwords suitable for development/testing only.

## Additional Resources

- [Zabbix Documentation](https://www.zabbix.com/documentation/current)
- [Zabbix Docker Images](https://hub.docker.com/u/zabbix)
- [Zabbix Create Hosts](https://docs.ansible.com/projects/ansible/latest/collections/community/zabbix/index.html)
- [snmpsim](https://pypi.org/project/snmpsim/)