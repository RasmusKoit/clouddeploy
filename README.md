# clouddeploy
Clouddeploy installation guide

##### Pictures of Clonedeploy in action
<details><summary>Pictures</summary>

<details><summary>Boot</summary>
 
![Clonedeploy boot](https://imgur.com/6umED6G.jpg "ClouneDeploy boot")

</details>

<details><summary>Login</summary>
 
![Clonedeploy login](https://imgur.com/CKVPQl6.jpg  "ClouneDeploy login")

</details>

<details><summary>Options</summary>
 
![Clonedeploy options](https://imgur.com/XP67P0x.jpg  "ClouneDeploy options")

</details>

</details>
## CloudDeploy workflow

![Workflow picture](https://i.imgur.com/jTKAoGj.jpg "CloudDeploy workflow")

Clients connect to PXE while booting and need to enter CloudDeploy service (clouddeploy/student for this example). At the same time they are given IPs from DHCP pool that is setup on the server. Next up after getting IPs and they have logged in they are promted to either upload or download their PC-s image. TFTP service runs on CloudDeploy server and communicates with Samba service to store those images. All the communications are routed with Shorewall. With this configuration not a lot of thought has been put to security since this example should only be used in a LAN with clients that you don't care about losing.

## Prerequisites

http://clonedeploy.org/

https://wiki.debian.org/HowTo/shorewall

* Debian derivative GNU/Linux server (Ubuntu 16.04 LTS was used for this tutorial) 
  * IP 192.168.0.1
* Clone Deploy server
  * IP 192.168.0.20
* Host machines
  * IP-s from 192.168.0.31-192.168.0.252

## Configuring gateway & DHCP & Firewall machine


### Setup interfaces

### IFUPDown config
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

### Netplan config

```
sudo nano /etc/netplan/01-netcfg.yaml
```

*Make sure you dont have 2 DHCP-s on one interface!*

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses: [192.168.7.109/24]
      gateway4: 192.168.7.254
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
    eth1:
      dhcp4: false
  bridges:
    br0:
      addresses: [192.168.0.1/24]
```
```
netplan generate
netplan apply
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
eth0 br0
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

```
sudo nano /etc/dhcp/dhcpd.conf
```

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

allow booting;
```

# Add a virtual machine

```
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils
```

Next up we want to change default settings to disable libvirt DHCP since we have our own dhcp server. We are also selecting our own bridge for default network.

```
virsh net-edit default
```

Edit this file and change the text to this one

You can use its current UUID instead.

```
<network>
  <name>default</name>
  <uuid>4f20d39a-32c7-4c63-866a-0600d82453aa</uuid> 
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
```

Save the settings and reload default config

```
virsh net-destory default
virsh net-start default
```

To check your current config matches the one you just wrote, you can write
```
virsh net-dumpxml default
```

Fix your iptables...

```
sudo iptables -t nat -A POSTROUTING -s 192.168.7.0/24 -j MASQUERADE
```

Now make sure you have a machine with virt-manager installed and make a connection to your current router to add a machine for Clone Deploy server

## Install Clone Deploy

Follow this guide

[Link to Clone Deploy](http://clonedeploy.org/docs/install-on-ubuntu/)

Now that all is done, you are good to go!
