# clouddeploy
Clouddeploy installation guide


## Prerequisites

http://clonedeploy.org/

https://wiki.debian.org/HowTo/shorewall

* Debian derivative GNU/Linux server (Ubuntu 16.04 LTS was used for this tutorial)
* Clone Deploy server
* Host machines

## Configuring gateway & DHCP & Firewall machine


### Setup interfaces

```
sudo nano /etc/network/interfaces
```


Make sure your configuration looks like this
```
# This is your WAN (outside network)
auto eth0
iface eth0 inet dhcp

# This is your LAN (Local network)
auto eth1
iface eth1 inet static
    address 192.168.0.1
    netmask 255.255.255.0
    network 192.168.0.0
```

Make sure eth0 and eth1 are the interfaces you need to use, by default those are correct


### Install shorewall

```
sudo apt update
sudo apt install shorewall
```

Now its time to change /etc/default/shorewall

```
sudo nano /etc/default/shorewall
```
Look for startup = 0
Change this to
```
startup = 1
```
save and exit

Now its time to enable shorewall service
```
systemctl enable shorewall
```

#### Configuring zones

We are making a new zones file
```
sudo nano /etc/shorewall/zones
```

Add these lines for your configuration


```
fw    firewall
net   ipv4
loc   ipv4
```


* fw is our router/dhcp/gateway/firewall
* net is network outside that fw gets internet
* loc is our internal network that host machines are connected to

Now that we have created our zones file, we have to bind interfaces to our zones

```
sudo nano /etc/shorewall/interfaces
```
Add these lines to your file

```
net eth0 detect dhcp,routefilter,tcpflags
loc eth1 detect dhcp
```

Next up we will set firewall policies
```
sudo nano /etc/shorewall/policy
```

Add next lines to your file

```
#SOURCE DEST  POLICY  LOGLEVEL
fw      all   ACCEPT
loc     all   ACCEPT
net     all   DROP    info
all     all   REJECT  info
```

Explanation:
    We are setting some basic firewall rules on how connections are handled.
    This file is read from top to bottom.
    
  Shorewall uses the first matching policy in the file.
  * Connections originated from fw have access to all other zones
  * Connections originated from loc have access to all other zones
  * Unknown connections from outside network are dropped and logged
  * Lastly if none of the above policies match, then connection is rejected ... and logged
  
#### Masquerading

Next we are making a masq file
```
sudo nano /etc/shorewall/masq
```
You only have to add one line here.

```
# Source destination
eth0 eth1
```

We are routing connections from eth0 to eth1

Now we have to allow forwarding ip connections

```
sudo nano /etc/shorewall/shorewall.conf
```
Look for IP_FORWARDING and change it to

```
IP_FORWARDING=On
```
save and quit

Its time to check if our shorewall configuration is correct

```
sudo shorewall check
```

If you get "Shorewall configuration verified" at the end, you may continue

```
sudo systemctl start shorewall
```

### Setting up DHCP and PXE

Install dhcpd
```
sudo apt install isc-dhcp-server
```

Now its time to configure your dhcp.
Make sure your file is edited like this

Uncomment this line
```
authoritative;
```
and add this block

```
subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.30 192.168.0.253;
  option domain-name-servers 1.1.1.1, 1.0.0.1;
  option subnet-mask 255.255.255.0;
  option routers 192.168.0.1;
  option broadcast-address 192.168.0.255
  default-lease-time 600;
  max-lease-time 7200;
  # TFTP Part for net booting
  filename "pxeboot.0";
  next-server 192.168.0.20;
}
```

## Install Clone Deploy

Follow this guide

[Link to Clone Deploy](http://clonedeploy.org/docs/install-on-ubuntu/)

Now that all is done, you are good to go!
