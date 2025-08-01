# HAProxy Setup with Modular Configuration and Cloudflare Integration

This guide provides a comprehensive walkthrough for installing and configuring HAProxy on a Debian-based system. The setup is designed for modularity, allowing for easy management of multiple services, and is specifically tailored for integration with Cloudflare.

## HAProxy Installation

First, install the HAProxy package using the system's package manager.

```bash
sudo apt update
sudo apt install haproxy
```

## Post-Installation: Modular Configuration

To enhance maintainability and scalability, this setup deviates from the default single `haproxy.cfg` file. Instead, we will configure HAProxy to load configuration snippets from a dedicated directory. This modular approach simplifies adding, removing, or modifying service configurations without altering a large, monolithic file.

### Create Configuration Directory

Create the directory that will hold the modular configuration files.

```bash
sudo mkdir -p /etc/haproxy/conf.d
```

### Systemd Service Modification

By default, the HAProxy `systemd` service unit is configured to load only `/etc/haproxy/haproxy.cfg`. We need to modify the service file to instruct HAProxy to also load all `.cfg` files from our new `/etc/