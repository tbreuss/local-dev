# LOCAL-DEV

This is my lightweight local development environment using dnsmasq, Docker, and Traefik running on MacOS Catalina.


## Goals

- Support for local development of multiple docker services with API interdependencies
- Ability to use *.test domain names from Mac host
- Ability to use same domain names inside Docker containers
- Support for HTTP and TCP routes 
- No more messing around in /etc/hosts


## Environment

- MacOS Catalina (10.15)
- dnsmasq (2.84)
- Docker Desktop for Mac (3.1.0)


## Solution

- Create persistent loopback interface for IP 10.254.254.254
- Install dnsmasq using IP 10.254.254.254 for nameserver and address target
- Launch Traefik and other containers using Docker Compose


## Included Docker Images

At the time of writing this repo includes configs for the following Docker images:

- [adminer:4.7](https://hub.docker.com/_/adminer)
- [containous/whoami:v1.5.0](https://hub.docker.com/r/containous/whoami)
- [mysql:5.7](https://hub.docker.com/_/mysql)
- [traefik:v2.4](https://hub.docker.com/_/traefik)


## Create Persistent loopback interface in macOS Catalina

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


## Install and configure dnsmasq

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


## Install Docker Desktop

<https://www.docker.com/products/docker-desktop>


## Launch Traefik and other containers

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
Hostname: 7c29d434f709
IP: 127.0.0.1
IP: 172.18.0.2
RemoteAddr: 172.18.0.5:49710
GET / HTTP/1.1
Host: whoami.test
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.88 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: de-DE,de;q=0.9,en-US;q=0.8,en;q=0.7
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.test
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: edcc555d7d77
X-Real-Ip: 172.18.0.1
~~~

Make cURL call from one docker container to another:

~~~bash
docker-compose exec adminer curl whoami.test
Hostname: 7c29d434f709
IP: 127.0.0.1
IP: 172.18.0.2
RemoteAddr: 172.18.0.5:49710
GET / HTTP/1.1
Host: whoami.test
User-Agent: curl/7.67.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.test
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: edcc555d7d77
X-Real-Ip: 172.18.0.1
~~~
 
Check the same after rebooting your Mac.


## Thanks

A big thanks to the authors of these helpful blog posts: 

- <https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a>
- <https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/>
- <https://blog.felipe-alfaro.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/>
