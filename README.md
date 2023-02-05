# LOCAL-DEV

> This is my lightweight local development environment using dnsmasq, Docker, and Traefik running on macOS Monterey.

- [Goals](#goals)
- [Prerequisites](#prerequisites)
- [Solution](#solution)
- [Included Docker Images](#included-docker-images)
- [Links](#links)


## Goals

- Support for local development of multiple docker services with API interdependencies
- Ability to use *.test domain names from Mac host
- Ability to use same domain names inside Docker containers
- Support for HTTP and TCP routes 
- Support for HTTPS (without self-signed certificate so far)
- No more messing around in /etc/hosts


## Prerequisites

This is my current set up:

- macOS Monterey (12.6)
- Homebrew (3.6)
- dnsmasq (2.88)
- Docker Desktop for Mac (4.16)

The instructions should also work with older versions.

## Solution

1. Create persistent loopback interface for IP 10.254.254.254
2. Install dnsmasq using IP 10.254.254.254 for nameserver and address target
3. Launch Traefik and other containers using Docker Compose


### 1. Create persistent loopback interface in macOS

Create a "launchd" daemon that configures an additional IPv4 address.

~~~bash
cat << EOF | sudo tee -a /Library/LaunchDaemons/ch.tebe.loopback1.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>ch.tebe.loopback1</string>
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
~~~

Launch service:

~~~bash
sudo launchctl load /Library/LaunchDaemons/ch.tebe.loopback1.plist
~~~

Make sure it works:

~~~bash
LaunchDaemons % sudo launchctl list | grep ch.tebe
-	0	ch.tebe.loopback1
~~~

Restart Mac and check ifconfig:

~~~bash
ifconfig lo0
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
	options=1203<RXCSUM,TXCSUM,TXSTATUS,SW_TIMESTAMP>
	inet 127.0.0.1 netmask 0xff000000
	inet6 ::1 prefixlen 128
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
	inet 10.254.254.254 netmask 0xff000000
	nd6 options=201<PERFORMNUD,DAD>
~~~


### 2. Install and configure dnsmasq

Install dnsmasq using Homebrew:

~~~bash
brew update # Always update Homebrew and the formulae first
brew install dnsmasq
~~~

Start dnsmasq service:

~~~bash
sudo brew services start dnsmasq
~~~

Open `/usr/local/etc/dnsmasq.conf` and add/uncomment the following line:

~~~bash
conf-dir=/usr/local/etc/dnsmasq.d,*.conf
~~~

Create custom conf file:

~~~bash
mkdir -p /usr/local/etc/dnsmasq.d
touch /usr/local/etc/dnsmasq.d/development.conf
~~~

Add routing rule for *.test domain names:

~~~bash
address=/.test/10.254.254.254 
~~~

Add custom resolver:

~~~bash
sudo mkdir /etc/resolver
~~~

Create a file `/etc/resolver/test` for the *.test domain names and add this line:

~~~bash
nameserver 10.254.254.254
~~~

Check that the resolver is registered.

~~~bash
scutil --dns
...
resolver #8
  domain   : test
  nameserver[0] : 10.254.254.254
  flags    : Request A records
  reach    : 0x00030002 (Reachable,Local Address,Directly Reachable Address)
...  
~~~

Check the dnsmasq setup:

~~~bash
ping -c 1 google.com # Make sure you can still access the outside world! 
ping -c 1 mysite.test
ping -c 1 my.other.site.test
~~~


### 3. Launch Traefik and other containers using Docker Compose

Install Docker Desktop:

<https://www.docker.com/products/docker-desktop>

Clone project from Github:

~~~
git clone https://github.com/tbreuss/local-dev.git
~~~

Start services using Docker Compose:

~~~
cd local-dev
docker-compose up
~~~

Check that everything works as expected.

Open `http://whoami.test` with your favorite browser. 
You should see something like:

~~~text
Hostname: eb7f1da188d7
IP: 127.0.0.1
IP: 172.18.0.5
RemoteAddr: 172.18.0.2:45232
GET / HTTP/1.1
Host: whoami.test
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: de-DE,de;q=0.9,en-US;q=0.8,en;q=0.7
Dnt: 1
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.test
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 73db93d4c8e8
X-Real-Ip: 172.18.0.1
~~~

Now, open `https://whoami.test` with your favorite browser.
The browser displays a NET::ERR_CERT_AUTHORITY_INVALID warning or similar, but lets you proceed to the website if you choose to.
You should see a similar output like above.

Make a cURL call from one docker container to another:

~~~bash
docker-compose exec adminer curl http://whoami.test
Hostname: eb7f1da188d7
IP: 127.0.0.1
IP: 172.18.0.5
RemoteAddr: 172.18.0.2:45238
GET / HTTP/1.1
Host: whoami.test
User-Agent: curl/7.80.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.test
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 73db93d4c8e8
X-Real-Ip: 172.18.0.1
~~~

Try the same using https:

~~~bash
docker-compose exec adminer curl --insecure https://whoami.test
~~~

You should see a similar output like above.

Don't forget to check the same after rebooting your Mac.


## Included Docker Images

At the time of writing this repo includes configs for the following Docker images:

- [adminer:4.8](https://hub.docker.com/_/adminer)
- [containous/whoami:v1.5](https://hub.docker.com/r/containous/whoami)
- [mailhog/mailhog:v1.0](https://hub.docker.com/r/mailhog/mailhog)
- [mysql:5.7](https://hub.docker.com/_/mysql)
- [traefik:v2.9](https://hub.docker.com/_/traefik)


## Links

Thanks to the authors of these helpful blog posts: 

- [Local Dev on Docker - Fun with DNS](https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a)
- [Use dnsmasq instead of /etc/hosts](https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/)
- [Persistent loopback interfaces in Mac OS X](https://felipealfaro.wordpress.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/)
- [Traefik Proxy 2.x and TLS 101](https://traefik.io/blog/traefik-2-tls-101-23b4fbee81f1/)
