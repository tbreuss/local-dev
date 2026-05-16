# Local Development Environment

> A lightweight local development environment for macOS using **dnsmasq**, **Docker**, and **Traefik** to enable seamless development of multiple services with API interdependencies.

## Table of Contents
- [Goals](#goals)
- [Prerequisites](#prerequisites)
- [Solution](#solution)
- [Included Docker Images](#included-docker-images)
- [References](#references)

## Goals

- **Multi-Service Support**: Facilitate local development involving multiple Docker services with complex API dependencies.
- **Unified DNS**: Use `*.test` domain names consistently on both the macOS host and inside Docker containers.
- **Routing**: Support for both HTTP and TCP routing.
- **HTTPS Ready**: HTTPS support (note: currently requires ignoring browser warnings for self-signed certificates).
- **Zero Maintenance**: Eliminate the need to manually edit `/etc/hosts`.

## Prerequisites

This setup was tested with the following versions but should be compatible with earlier releases:

- **OS**: macOS Tahoe (26.0)
- **Package Manager**: Homebrew (4.6)
- **DNS Server**: dnsmasq (2.91)
- **Container Engine**: Docker Desktop for Mac (4.48)

## Solution

The setup consists of three main steps:
1. Create a persistent loopback interface for IP `10.254.254.254`.
2. Configure dnsmasq to route `*.test` domains to this IP.
3. Launch Traefik and other containers via Docker Compose.

### 1. Create a Persistent Loopback Interface on macOS

We will create a `launchd` daemon to assign an additional IPv4 address (`10.254.254.254`) to the loopback interface (`lo0`), ensuring it persists after reboots.

**Create the Launch Daemon:**

```bash
cat << EOF | sudo tee -a /Library/LaunchDaemons/com.github.tbreuss.local-dev.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.github.tbreuss.local-dev</string>
    <key>ProgramArguments</key>
    <array>
        <string>/sbin/ifconfig</string>
        <string>lo0</string>
        <string>alias</string>
        <string>10.254.254.254</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
EOF
```

**Load the Service:**

```bash
sudo launchctl load /Library/LaunchDaemons/com.github.tbreuss.local-dev.plist
```

**Verify the Service:**

```bash
sudo launchctl list | grep local-dev
# Expected output: -	0	com.github.tbreuss.local-dev
```

**Check the Interface (after a reboot):**

```bash
ifconfig lo0
# Look for: inet 10.254.254.254 netmask 0xff000000
```

### 2. Install and Configure dnsmasq

**Install dnsmasq via Homebrew:**

```bash
brew update
brew install dnsmasq
sudo brew services start dnsmasq
```

**Configure dnsmasq:**

Open `/usr/local/etc/dnsmasq.conf` and ensure the following line is present (uncommented):

```bash
conf-dir=/usr/local/etc/dnsmasq.d,*.conf
```

Create a custom configuration file:

```bash
mkdir -p /usr/local/etc/dnsmasq.d
touch /usr/local/etc/dnsmasq.d/development.conf
```

Add the routing rule to direct all `*.test` domains to our loopback IP:

```bash
address=/.test/10.254.254.254
```

**Configure macOS Resolver:**

Create a resolver file for the `.test` TLD so macOS knows to query dnsmasq for these domains:

```bash
sudo mkdir -p /etc/resolver
echo "nameserver 10.254.254.254" | sudo tee /etc/resolver/test
```

**Verify the Setup:**

Check that the resolver is registered:

```bash
scutil --dns
# Look for a section "resolver #X" where domain is "test" and nameserver is "10.254.254.254"
```

**Test DNS resolution:**

```bash
# Ensure external domains still work
ping -c 1 google.com

# Test local resolution
ping -c 1 mysite.test
ping -c 1 my.other.site.test
```

### 3. Launch Traefik and Containers via Docker Compose

**Install Docker Desktop** (if not already installed):
[Download Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)

**Clone the Project:**

```bash
git clone https://github.com/tbreuss/local-dev.git
cd local-dev
```

**Start Services:**

```bash
docker-compose up
```

**Verification:**

1. Open `http://whoami.test` in your browser.
   You should see output similar to:
   ```text
   Hostname: eb7f1da188d7
   IP: 127.0.0.1
   IP: 172.18.0.5
   RemoteAddr: 172.18.0.2:45232
   GET / HTTP/1.1
   Host: whoami.test
   ...
   X-Forwarded-Host: whoami.test
   X-Forwarded-Proto: http
   ```

2. Open `https://whoami.test`.
   The browser will likely show a certificate warning (e.g., `NET::ERR_CERT_AUTHORITY_INVALID`). You can proceed past this warning. The output should be similar to the HTTP version.

3. **Test Inter-Container Communication**:
   Execute a request from one container to another (e.g., from `adminer`):
   ```bash
   docker-compose exec adminer curl http://whoami.test
   ```

   For HTTPS (using `--insecure` to skip cert validation):
   ```bash
   docker-compose exec adminer curl --insecure https://whoami.test
   ```

**Important:** Always verify that DNS resolution and container routing still work after rebooting your Mac.

## Included Docker Images

At the time of writing, this repository includes configurations for the following images:

- [adminer:5.4.1](https://hub.docker.com/_/adminer)
- [containous/whoami:v1.5.0](https://hub.docker.com/r/containous/whoami)
- [mailhog/mailhog:v1.0.1](https://hub.docker.com/r/mailhog/mailhog)
- [mysql:8.4.7](https://hub.docker.com/_/mysql)
- [traefik:v3.5.3](https://hub.docker.com/_/traefik)

## References

Special thanks to the authors of the following guides which inspired this setup:

- [Local Dev on Docker - Fun with DNS](https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a)
- [Use dnsmasq instead of /etc/hosts](https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/)
- [Persistent loopback interfaces in Mac OS X](https://felipealfaro.wordpress.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/)
- [Traefik Proxy 2.x and TLS 101](https://traefik.io/blog/traefik-2-tls-101-23b4fbee81f1/)
