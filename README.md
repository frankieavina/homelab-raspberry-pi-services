# Raspberry Pi 5 – Homelab Infrastructure Services

A Raspberry Pi 5-based infrastructure server built to provide essential network services, DNS filtering, network monitoring, and print management for my personal homelab.

This project is part of my homelab portfolio, where I practice Linux system administration, networking, Docker, infrastructure monitoring, and IT troubleshooting in an environment modeled after a small enterprise network.

## Project Overview

The Raspberry Pi operates as an independent network services server. It provides DNS filtering through Pi-hole, infrastructure monitoring through Uptime Kuma, and network printing through CUPS.

The goal is to keep essential network services separate from my virtualization and NAS systems.

### Hardware & Environment

| Component | Configuration |
|---|---|
| Device | Raspberry Pi 5 |
| Operating System | Debian GNU/Linux 13 (Trixie) |
| Architecture | ARM64 |
| Hostname | FrankiesPi |
| Static DHCP Reservation | 192.168.1.169 |
| Network Connection | Wired Ethernet |
| Gateway | UniFi Cloud Gateway Ultra |
| Gateway IP | 192.168.1.1 |

## Network Architecture

The Raspberry Pi is connected to a managed network switch in the homelab server room.

```text
                 Internet
                    |
              Spectrum Modem
                    |
          UniFi Cloud Gateway Ultra
                192.168.1.1
                    |
              Netgear GC110P
                    |
             Raspberry Pi 5
              192.168.1.169
                    |
       +------------+------------+
       |            |            |
     Pi-hole    Uptime Kuma     CUPS
      DNS       Monitoring     Printing
```

The UniFi Cloud Gateway Ultra remains responsible for routing, firewall policies, and DHCP. Pi-hole provides DNS filtering for network clients.

## Services Deployed

### 1. Pi-hole – DNS Filtering

Pi-hole provides network-wide DNS filtering and ad blocking.

**Configuration:**
- Installed natively on Debian.
- DNS service listening on port 53.
- Google DNS configured as upstream resolver (`8.8.8.8`).
- UniFi network configured to distribute the Raspberry Pi as its DNS server.
- Default StevenBlack blocklist enabled.

**Web Interface:**

`http://192.168.1.169/admin`

**Verification:**
- Confirmed DNS resolution using `nslookup`.
- Confirmed blocked domains resolve to `0.0.0.0` and `::`.
- Verified Pi-hole DNS service was listening on port 53.

### 2. Uptime Kuma – Infrastructure Monitoring

Uptime Kuma provides a centralized monitoring dashboard for infrastructure devices and network services.

**Deployment:**
- Docker Engine
- Docker Compose
- Persistent application data stored locally.
- SQLite database.
- Automatic container restart policy.

**Web Interface:**

`http://192.168.1.169:3001`

**Docker project location:**

`/opt/uptime-kuma`

**Docker Compose configuration:**

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
```

**Configured Monitors:**

| Monitor | Monitoring Method |
|---|---|
| UniFi Cloud Gateway Ultra | Ping |
| Raspberry Pi 5 | Ping |
| Proxmox M720q | Ping |
| Unraid Server | Ping |
| Pi-hole DNS | DNS |
| Uptime Kuma Web UI | HTTP |
| CUPS Print Server | HTTP |
| Raspberry Pi System Health | Push (CPU temperature, RAM, disk, and load) |

#### Host-Level System Health Reporting

A Python health-check script runs through a systemd timer every five minutes and sends CPU temperature, RAM utilization, root filesystem utilization, and normalized CPU load to a Uptime Kuma Push monitor. Configurable thresholds cause the report to change from UP to DOWN when a warning condition is detected.

**Documentation:** [Raspberry Pi System Health Monitoring](docs/raspberry-pi-system-health-monitoring.md)

The Push URL is stored locally in a protected environment file and is not included in this repository.


## 3. Automated Backups & Disaster Recovery

The Raspberry Pi uses Restic to create encrypted, incremental backups on an SMB share hosted by the Unraid NAS. A systemd automount keeps the NAS optional during Pi startup; the backup script checks NAS availability and skips cleanly when Unraid is offline.

The weekly workflow refreshes a consistent Uptime Kuma SQLite backup, a Pi-hole Teleporter configuration export (with ZIP integrity validation), CUPS configuration, and selected service/system configuration files before creating a Restic snapshot. A systemd timer schedules the job every Sunday at 2:00 AM Pacific. The repository has been integrity-checked, and a restore test successfully recovered 45 files and directories.

**Documentation:** [Raspberry Pi Automated Backups and Disaster Recovery](docs/raspberry-pi-automated-backups.md)

## 4. CUPS – Print Server

CUPS (Common UNIX Printing System) provides network printing functionality.

**Configuration:**
- Installed and running as a native Linux service.
- Enabled to start automatically.
- Web interface available on port 631.

**Web Interface:**

`http://192.168.1.169:631`

**Verification:**
- Confirmed `cups.service` was active and running.
- Confirmed the web interface responded successfully.

## Monitoring & Troubleshooting Exercise

### Simulated Service Outage and Recovery

An intentional Uptime Kuma outage was performed to test service recovery and monitoring behavior.

**Procedure:**

1. Stopped the Uptime Kuma Docker container.
2. Observed the monitoring system detect the service outage.
3. Restarted the container using Docker Compose.
4. Verified the container returned to a healthy state.
5. Confirmed the Uptime Kuma monitor returned to UP.

**Commands used:**

Stop the service:

```bash
cd /opt/uptime-kuma
sudo docker compose stop uptime-kuma
```

Verify container status:

```bash
sudo docker compose ps
```

Restart the service:

```bash
sudo docker compose start uptime-kuma
```

Verify recovery:

```bash
sudo docker compose ps
```

**Outcome:** Successful outage detection, service restoration, and monitoring recovery.

<img width="1406" height="832" alt="Screenshot 2026-11-19 at 10 40 52 AM" src="https://github.com/user-attachments/assets/c0f06418-51ba-40d7-a94f-94001d68f09a" />


## Skills Demonstrated

- Linux system administration (Debian)
- Raspberry Pi administration
- Network troubleshooting
- DNS configuration and troubleshooting
- Pi-hole deployment and administration
- Docker Engine installation and management
- Docker Compose deployment
- Container persistence and restart policies
- Uptime Kuma monitoring configuration
- HTTP, DNS, ICMP, and Push monitoring
- Python-based Linux resource monitoring
- systemd oneshot services and timers
- SMB/CIFS mounts and systemd automounts
- Bash automation and error handling
- Restic encrypted incremental backups
- SQLite online database backups
- Backup integrity verification and restore testing
- Disaster recovery documentation
- CUPS print server administration
- Service availability testing
- Incident simulation and recovery
- Technical documentation

## Security Considerations

- DHCP remains managed by the UniFi gateway.
- Pi-hole provides DNS filtering without taking over DHCP responsibilities.
- Uptime Kuma is intended for local network access.
- No credentials, passwords, API keys, or other secrets should be committed to this repository.
- Administrative interfaces should not be exposed directly to the public internet.

## Future Improvements

- [x] Configure automatic security updates.
- [x] Review Linux services and exposed network ports.
- [x] Implement regular backups of Pi-hole and Uptime Kuma configurations.
- [x] Add system resource monitoring.
- [ ] Configure monitoring notifications if needed.
- [ ] Document additional troubleshooting scenarios.
- [ ] Add network architecture diagrams and screenshots.
- [x] Document and test a disaster recovery procedure.


## Project Status

**Current status:** Core infrastructure services deployed and verified.

This project is actively maintained and will evolve as additional infrastructure, monitoring, security, and automation capabilities are implemented.

---
