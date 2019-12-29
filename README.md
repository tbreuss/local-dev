# README

## Persistent loopback interfaces in macOS Catalina

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

Start the service up:

~~~bash
sudo launchctl load /Library/LaunchDaemons/ch.tebe.loopback1.plist
~~~

Make sure it did work:

~~~bash
LaunchDaemons % sudo launchctl list | grep ch.tebe
-	0	ch.tebe.loopback1
~~~

Restart Macbook:

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


## Links

- <https://docs.traefik.io/>
- <https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a>
- <https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/>
- <https://blog.felipe-alfaro.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/>
