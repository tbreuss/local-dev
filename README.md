# LOCAL-DEV

> This is my lightweight local development environment using dnsmasq, Docker, and Traefik running on macOS Monterey.

- [Goals](#goals)
- [Prerequisites](#prerequisites)
- [Solution](#solution)
- [Included Docker Images](#included-docker-images)
- [Links](#links)

## Goals

- Support for local development of multiple docker services with API interdependencies
- Ability to use *.test domain names from macOS host
- Ability to use same domain names inside Docker containers
- Support for HTTP and TCP routes 
- Support for HTTPS (without self-signed certificate so far)
- No more messing around in /etc/hosts

## Prerequisites

This is my current set up:

- macOS Tahoe (26.0)
- Homebrew (4.6)
- dnsmasq (2.91)
- Docker Desktop for Mac (4.48)

The instructions should also work with older versions.

## Solution

### Step 1: Install and configure Dnsmasq

Install dnsmasq using Homebrew:

~~~bash
brew update # Always update Homebrew and the formulae first
brew install dnsmasq
~~~

Setup domain configuration for our `*.test` domain:

~~~bash
echo 'address=/.test/127.0.0.1' >> $(brew --prefix)/etc/dnsmasq.conf
~~~

Start dnsmasq service:

~~~bash
sudo brew services start dnsmasq
~~~

Check whether the dnsmasq service is running:

~~~bash
sudo brew services list         
...
Name    Status  User File
dnsmasq started root /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
~~~

Perform a dig command to query your local dnsmasq instance and verify that the configuration was successful:

~~~bash
dig mysite.test @127.0.0.1
~~~

### Step 2: Create a DNS resolver and test setup

Create a resolver directory if it doesn't already exist:

~~~bash
sudo mkdir -v /etc/resolver
~~~

Add our dnsmasq nameserver to resolvers:

~~~bash
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/test'
~~~

Check whether the resolver has been successfully registered:

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

Check whether external links are still being resolved successfully:

~~~bash
ping -c 1 google.com # Make sure you can still access the outside world! 
~~~

Now check if dnsmasq handles all requests on the `.test` domain:

~~~bash
ping -c 1 mysite.test
ping -c 1 my.other.site.test
~~~

This should return replies from `127.0.0.1`.

### Step 3: Launch Traefik and the other containers using Docker Compose

Install Docker Desktop:

<https://www.docker.com/products/docker-desktop>

Clone project from GitHub:

~~~
git clone https://github.com/tbreuss/local-dev.git
~~~

Start services using Docker Compose:

~~~
cd local-dev
docker compose up
~~~

Check that everything is working as expected.

Open http://whoami.test with your preferred browser.
You should see something like this:

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

Now, open https://whoami.test with your preferred browser.
The browser displays a NET::ERR_CERT_AUTHORITY_INVALID warning or similar, but lets you proceed to the website if you choose to.
You should see a similar output like above.

Make a cURL call from one docker container to another:

~~~bash
docker compose exec adminer curl http://whoami.test
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
docker compose exec adminer curl --insecure https://whoami.test
~~~

You should see a similar output like above.

Don't forget to check the same after rebooting your Mac.


## Included Docker Images

At the time of writing this repo includes configs for the following Docker images:

- [adminer:5.4.1](https://hub.docker.com/_/adminer)
- [containous/whoami:v1.5.0](https://hub.docker.com/r/containous/whoami)
- [mailhog/mailhog:v1.0.1](https://hub.docker.com/r/mailhog/mailhog)
- [mysql:8.4.7](https://hub.docker.com/_/mysql)
- [traefik:v3.5.3](https://hub.docker.com/_/traefik)


## Related Links

- [Local Dev on Docker - Fun with DNS](https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a)
- [Use dnsmasq instead of /etc/hosts](https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/)
- [Persistent loopback interfaces in Mac OS X](https://felipealfaro.wordpress.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/)
- [Traefik Proxy 2.x and TLS 101](https://traefik.io/blog/traefik-2-tls-101-23b4fbee81f1/)
- [Install dnsmasq and configure for *.dev.local domains](https://gist.github.com/davebarnwell/c408533d608bfe24f4f5)
- [Setup Automatic Local Domains with Dnsmasq on macOS Ventura](https://allanphilipbarku.medium.com/setup-automatic-local-domains-with-dnsmasq-on-macos-ventura-b4cd460d8cb3)
