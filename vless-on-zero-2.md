### Problem
For some reason 😒 we need to provide VPNed wifi-network
for our device using VLESS technology.

### Requirements
raspberry pi zero 2 W
tp link wn823n v3
a couple of short USB type A male to microUSB male cables
powerbank (optional)
VPS with static ip in allowed-to-connect subnet from one of approved 😏 hosting providers

### What will we get?
For our has-no-admin-access device with corporate-network-vpn
ability to get connection to global internet in case of blocking everything except several approved subnets

### Links
https://www.raspberrypi.com/software/  
https://github.com/Mange/rtl8192eu-linux-driver 

### Instructions
#### OS and drivers
Download raspberry pi imager.  
Install 64-bit no-desktop (lite) debian bookworm (legacy) system (provide info about wifi-connection, enable ssh with login/pass).  
go to rtl8192eu-linux-driver from github  
follow DKMS instructions  
in the point 3 do changes for arm64.  
After that step you have to get 2 wlan (wlan0 and wlan1) in "ifconfig -a".  

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

#### Problem with interface configuration
And at this point we had trouble with isc-dhcp-server. It just won't start.
Dunno why but in boorworm /etc/network/interfaces just doesn't work so we need a workaround.

nano /usr/local/bin/fix-wlan1-on-boot.sh
```
#!/bin/bash
sudo ifconfig wlan1 192.168.99.1 netmask 255.255.255.0
sudo service isc-dhcp-server restart
sudo service tun2socks start
exit 0
```
chmod +x /usr/local/bin/fix-wlan1-on-boot.sh

nano /etc/systemd/system/fix-wlan1-on-boot.service
```
[Unit]
Description=Fix wlan1 for access point mode on boot
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/fix-wlan1-on-boot.sh
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
```

sudo systemctl daemon-reload
sudo systemctl start fix-wlan1-on-boot.service
sudo systemctl enable fix-wlan1-on-boot.service

#### Add another supplier wifi
check the content of /etc/NetworkManager/system-connections/ and add new connection-file

#### Setup xray interface
##### prepare system
```
apt update
apt -y install git curl unzip wireguard iptables iptables-persistent wget tcpdump
git clone https://github.com/Letowski/hardware-vpn.git
cd hardware-vpn
touch info.txt
```

##### set envs received from server's setup
```
export IP_EXIT=xxx
export XRAY_SITE=xxx
export XRAY_UUID=xxx
export XRAY_PRIVATE=xxx
export XRAY_PUBLIC=xxx
export XRAY_SHORT=xxx
```

##### install tun2socks
```
wget https://github.com/xjasonlyu/tun2socks/releases/download/v2.6.0/tun2socks-linux-arm64.zip 
unzip tun2socks-linux-arm64.zip
chmod +x ./tun2socks-linux-arm64
mv ./tun2socks-linux-arm64 /usr/local/bin/tun2socks
rm tun2socks-linux-arm64.zip
cp vless-on-zero-2/tun2socks.service /etc/systemd/system/tun2socks.service
service tun2socks start
```

##### install xray
```
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install -u root
```

##### configure xray
```
rm /usr/local/etc/xray/config.json
sed -i -e "s/IP_EXIT/$IP_EXIT/g" vless-on-zero-2/config.json
sed -i -e "s/XRAY_UUID/$XRAY_UUID/g" vless-on-zero-2/config.json
sed -i -e "s/XRAY_SITE/$XRAY_SITE/g" vless-on-zero-2/config.json
sed -i -e "s/XRAY_PUBLIC/$XRAY_PUBLIC/g" vless-on-zero-2/config.json
sed -i -e "s/XRAY_SHORT/$XRAY_SHORT/g" vless-on-zero-2/config.json
cp vless-on-zero-2/config.json /usr/local/etc/xray/config.json
systemctl restart xray
timeout 5s systemctl status xray
```