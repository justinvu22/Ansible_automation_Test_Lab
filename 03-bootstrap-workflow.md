# TO5 Infrastructure Rebuild Runbook

## Purpose

This document describes the complete process to rebuild the TO5 infrastructure from scratch.

The goal is to achieve a reproducible environment where all virtual machines are configured through Ansible.

---

# Phase 0 — Prepare Infrastructure Repository

On MacBook:

```bash
git clone <TO5 repository>
cd to5-infrastructure/ansible
```

Verify:

```bash
ansible --version
ssh -V
```

---

# Phase 1 — Install Proxmox VE

Manual steps:

* Install Proxmox VE.
* Configure storage.
* Configure network bridge (`vmbr0`).
* Configure management IP.
* Configure DNS and gateway.
* Update Proxmox packages.

Verify:

```bash
ip addr
ping 8.8.8.8
```

---

# Phase 2 — Create Ubuntu Golden Template

Create Ubuntu 24.04 LTS VM.

The template should include:

## Required packages

* OpenSSH Server
* cloud-init
* qemu-guest-agent
* Python3
* sudo

## User configuration

Create:

```
admin1
```

Configure:

* sudo privileges
* MacBook SSH public key in:

```
/home/admin1/.ssh/authorized_keys
```

## Network

Configure:

* DHCP or cloud-init network configuration.
* Correct DNS server.
* Correct default gateway.

---

# Phase 3 — Clone Virtual Machines

Create required VMs:

```
teleport01
prom01
ansible01

nginx01
nginx02
nginx03

haproxy01
haproxy02

dns01
```

Assign:

* CPU
* Memory
* Disk
* Hostname
* Static IP (if used)

---

# Phase 4 — Verify Initial SSH Access

From MacBook:

Test all servers:

```bash
ssh admin1@<server-ip>
```

Example:

```bash
ssh admin1@192.168.100.68
```

All VMs must be reachable before running Ansible.

---

# Phase 5 — Configure Ansible Inventory

Update:

```
ansible/inventory/hosts.ini
```

Verify:

* Hostnames.
* IP addresses.
* Groups:

  * teleport_server
  * teleport_nodes
  * web_servers
  * load_balancers
  * monitoring
  * monitor_targets

Test:

```bash
ansible all -m ping
```

Expected:

```
SUCCESS
```

---

# Phase 6 — Common Operating System Bootstrap

Run:

```bash
ansible-playbook playbooks/bootstrap-common.yml
```

This will configure:

* Base packages.
* System updates.
* Time synchronization.
* qemu-guest-agent.
* Python environment.
* Basic operating system configuration.

---

# Phase 7 — Deploy Teleport Control Plane

Target:

```
teleport01
```

Run:

```bash
ansible-playbook playbooks/install-teleport-server.yml
```

Verify:

From Mac:

```bash
tsh login --proxy=<teleport-proxy> --user=<user>
```

Check:

```bash
tsh ls
```

Expected:

```
teleport01
```

---

# Phase 8 — Generate Teleport Node Join Token

SSH into:

```
teleport01
```

Create token:

```bash
sudo tctl tokens add --type=node
```

Example:

```
Token: abc123xyz
```

---

# Phase 9 — Deploy Teleport Nodes

On Mac:

Export the token:

```bash
export TELEPORT_JOIN_TOKEN=abc123xyz
```

Deploy:

```bash
ansible-playbook playbooks/install-teleport-node.yml
```

Verify:

```bash
tsh ls
```

Expected nodes:

```
teleport01
nginx01
nginx02
nginx03
haproxy01
haproxy02
dns01
prom01
ansible01
```

---

# Phase 10 — Deploy Node Exporters

Run:

```bash
ansible-playbook playbooks/install-node-exporter.yml
```

Verify:

Example:

```bash
curl http://nginx01:9100/metrics
```

Expected:

```
# HELP ...
# TYPE ...
```

---

# Phase 11 — Deploy Prometheus Server

Target:

```
prom01
```

Run:

```bash
ansible-playbook playbooks/install-prometheus.yml
```

Verify:

Open:

```
http://prom01:9090
```

Check:

```
Status → Targets
```

Expected:

```
UP
```

for all nodes.

---

# Phase 12 — Final Verification

## Teleport

From Mac:

```bash
tsh status
tsh ls
tsh ssh admin1@nginx01
```

---

## Monitoring

Verify:

* Prometheus is active.
* All Node Exporters are reachable.
* Metrics are being collected.

---

## System Services

Check:

```
teleport
node_exporter
prometheus
qemu-guest-agent
chrony
```

---

# Recovery Philosophy

The infrastructure should follow this principle:

```
Proxmox
   |
Ubuntu Golden Template
   |
Clone Virtual Machines
   |
MacBook Ansible Control Node
   |
Bootstrap
   |
Teleport
   |
Monitoring
   |
Applications
```

No manual server configuration should be required after SSH bootstrap.

Any permanent change must be implemented through:

* Ansible inventory.
* Variables.
* Playbooks.
* Templates.
* Git version control.

---

# Future Improvements

Possible future additions:

* Grafana dashboard.
* Alertmanager.
* Ansible Vault for secrets.
* CI/CD pipeline for infrastructure validation.
* Automatic Proxmox VM provisioning using Terraform.
* Dedicated Ansible automation server.
* Backup and disaster recovery procedures.

```
```
