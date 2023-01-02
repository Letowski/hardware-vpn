### Problem

You have a device with not-root-access (maybe there is an antivirus on it).  
For some reason you have some requirements for the network you can connect to.  
The device contains software that supervise the connected network (and you don't have permissions to disable it).  
For example: laptop belongs to your employer with security requirements of working from certain country.  
You don't have enough access to disable installed supervision software but want to travel abroad with it.

### Requirements

For that build you need:  
raspberry pi (in my case I have used 3B) with usbA (general type) and preinstalled wifi-module  
tp link wn823n v3 (maybe v2 can be used too)  
Maybe you can do that with Raspberry Pi Zero W + usb adapter typeA->miniUsb  
After all you have to get access for working wireguard server (as endpoint of your network)  

### What will you get?

Small box. It can be powered from small (10mAh+) power-bank.  
It can receive wifi from your regular supplier (for ex. from your phone)  
and share vpn-ed wifi-network to your target device.  

### Links

https://www.raspberrypi.com/software/  
https://github.com/Mange/rtl8192eu-linux-driver  

### Instructions
#### OS and drivers

Download raspberry pi imager.  
Install latest 32-bit no-desktop system (provide info about wifi-connection and ssh credentials).  
go to rtl8192eu-linux-driver from github  
follow DKMS instructions  
in the point 3 do changes for raspberry pi.  
After that step you have to get 2 wlan (wlan0 and wlan1) in "ifconfig -a".  
Both of them have to be connected to your supplier-wifi.  

#### Wireguard

apt install wireguard  
/etc/wireguard/wg0.conf (change literals in <> to values of you wireguard-server)  

```
[Interface]
PrivateKey = <PrivateKey>
Address = 10.11.11.2/32, fd42:42:42::2/128
DNS = 8.8.8.8, 8.8.4.4

[Peer]
PublicKey = <PublicKey>
PresharedKey = <PresharedKey>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <Endpoint>
```

systemctl enable wg-quick@wg0  
wg-quick up wg0  
This will add wg0 in "ifconfig -a" and will route all traffic inside raspberry through vpn.  
Check this by calling "ping <your-favorite-site.com>". Packets must go without loss.  

#### Providing IPs 

apt install isc-dhcp-server iptables-persistent hostapd isc-dhcp-server -y  
/etc/network/interfaces (wlan1 will be 192.168.99.1 and sharing wifi with ips 192.168.99.*)  

```
allow-hotplug wlan1
iface wlan1 inet static
address 192.168.99.1
netmask 255.255.255.0
```

/etc/default/isc-dhcp-server (wlan1 will set ipV4 for other devices)

```
DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
INTERFACESv4="wlan1"
INTERFACESv6=""
```

/etc/dhcp/dhcpd.conf (new ips will be from 192.168.99.100 to 192.168.99.150)

```
ddns-update-style none;
log-facility local7;
authoritative;
subnet 192.168.99.0 netmask 255.255.255.0 {
range 192.168.99.100 192.168.99.150;
option broadcast-address 192.168.99.255;
option routers 192.168.99.1;
default-lease-time 60000;
max-lease-time 72000;
option domain-name "polygon.local";
option domain-name-servers 8.8.8.8;
}
```

/etc/sysctl.conf uncomment line or add in the end

```
net.ipv4.ip_forward = 1
```

/etc/dhcpcd.conf in the end of file add (disable wpa_supplicant on wlan1)

```
interface wlan1
nohook wpa_supplicant
```

/etc/hostapd/hostapd.conf config of your new wifi (change literals in <> to wifi name and password)

```
interface=wlan1
ssid=<wifi_name>
hw_mode=g
channel=3
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=<password>
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

check your build with "hostapd /etc/hostapd/hostapd.conf"  
if there is no errors invoke  
sudo systemctl unmask hostapd  
sudo systemctl enable hostapd  
sudo systemctl start hostapd  

#### Iptables

iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE  
iptables -A FORWARD -i wg0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT  
iptables -A FORWARD -i wlan1 -o wg0 -j ACCEPT  
netfilter-persistent save  
netfilter-persistent reload  

After all reboot raspberry. It has to start all services after startup.

### Privacy

Make sure that geolocation sensor is disabled on your device.  
Manage your device to forget all previous wifi-networks (to avoid connecting to wrong one).  
In the other way your privacy may be compromised.  

### We need to go cheaper

You can use orange pi zero lts.  
Just use eth0 (network wire) as output interface.  
You don't have to install hostapd.  
/etc/default/isc-dhcp-server set 'INTERFACESv4="wlan1"'  
in the iptables-commands change wlan1 to eth0  
in the /etc/network/interfaces set "iface wlan1 inet static"  
That case is more private because you can disable wifi-card  
on your target device and avoid getting information  
about environment by scanning nearly wifi-networks.  






