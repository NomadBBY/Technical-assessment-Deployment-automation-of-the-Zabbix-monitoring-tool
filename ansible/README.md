# Zabbix Automation

Ansible role to automate Zabbix monitoring configuration.

## Structure

```
zabbix-automation/
├── defaults/           # Default variables
├── tasks/             # Task files
│   ├── main.yml                  # Main task orchestration
│   ├── set_credentials.yml       # Set API credentials
│   ├── add_zabbix_agents.yml     # Add Zabbix server and proxy agents
│   ├── add_zabbix_proxy.yml      # Configure Zabbix proxy
│   └── add_snmp_hosts.yml        # Add SNMP devices
├── inventory/         # Inventory files
│   ├── docker_containers.docker.yml
│   └── docker_inventory.yml
└── vars/              # Role variables
```

## Usage

### Run all automation tasks

From the ansible directory, run:

```bash
cd /Users/taacear3/WORK/ZABBIX/ansible
ansible-playbook site.yml
```

This will execute all tasks in order:
1. Set API credentials
2. Add Zabbix agents (server and proxy)
3. Configure Zabbix proxy
4. Add SNMP hosts

### Run specific tasks

You can also run individual task files by creating a custom playbook:

```yaml
---
- name: Run specific task
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Include credentials
      ansible.builtin.include_role:
        name: zabbix-automation
        tasks_from: set_credentials

    - name: Add only SNMP hosts
      ansible.builtin.include_role:
        name: zabbix-automation
        tasks_from: add_snmp_hosts
```

## Configuration

Edit `defaults/main.yml` to customize:
- API credentials
- Server/proxy IP addresses
- SNMP device details

## Requirements

- Ansible collection: `community.zabbix`
- Docker containers running (Zabbix server, agents, SNMP simulators)
