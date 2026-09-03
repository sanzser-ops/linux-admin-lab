# Architecture

This document describes the architecture of the Linux Production Operations Lab.

## Lab Environment

The lab runs on local virtual machines using Oracle VirtualBox.

The environment consists of two Linux servers:

| Server | Operating System | Private IP |
|---|---|---|
| rhel-server-01 | Red Hat Enterprise Linux 10.2 | 192.168.56.10 |
| ubuntu-server-01 | Ubuntu Server 26.04.1 LTS | 192.168.56.11 |

## Network Architecture

Each server has two network interfaces.

### NAT Interface

The NAT interface provides outbound Internet connectivity for:

- Operating system updates
- Package installation
- External repository access

### Host-Only Interface

The Host-Only interface provides isolated communication between the lab servers.

Network:

```text
192.168.56.0/24
```

VirtualBox Host-Only adapter:

```text
192.168.56.1
```

Server addresses:

```text
RHEL:
192.168.56.10

Ubuntu:
192.168.56.11
```

The Host-Only network is not used as an Internet gateway.

## Server Connectivity

The servers can communicate with each other using the private network.

Connectivity has been verified using:

- ICMP (ping)
- SSH

SSH connectivity has been tested in both directions:

```text
rhel-server-01 → ubuntu-server-01
ubuntu-server-01 → rhel-server-01
```

## SSH Access

SSH is enabled and configured to start automatically on both servers.

RHEL:

```text
sshd.service
active
enabled
```

Ubuntu:

```text
ssh.service
active
enabled
```

## Virtualization

The environment is hosted locally using Oracle VirtualBox.

The lab does not require permanently running cloud infrastructure.

## Design Goals

The architecture is intentionally simple at this stage.

Future iterations will introduce:

- Ansible automation
- Containerized services
- Monitoring
- Alerting
- Security hardening
- Incident simulations
- Operational runbooks
