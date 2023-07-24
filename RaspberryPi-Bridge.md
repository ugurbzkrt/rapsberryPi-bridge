# Raspberry Pi 4 Model B İle WiFi Ethernet Köprüleme

Burada yapılacak olan işlem Wifi kartı bulunmayan bir masaüstü bilgisayarı bir Raspberry Pi aracılığıyla bir wifi ağına bağlamaktır. Wifi olmayan bilgisayar ile Wifiağı arasında bir köprü olarak bir Raspberry Pi 4 Model B kullanacağız.

Raspberry Pi 4, Wifi’ye bağlanacak ve bağlantısını Ethernet kullanarak diğer bilgisayara aktaracak. Raspberry Pi 4 Model B ac Wifi’ye sahip olduğu için hız açısından da sıkıntı çekmeyeceksiniz 100mbitleri görmek mümkün.

Ben kurulum için raspbian kullanacağım ama isteyen olursa dietpi kullanabilir. Her ikisi de Debian zaten sorun yaşamazsınız.

Öncesinde Wifi’ye bağlayalım cihazı, boot aşamasında bir işlem yapmamışsanız aşağıdaki şekilde ayarlayabilirsiniz.(bu işlemi dietpi üzerinde boot yapılandırması ile çözebilirsiniz.)

```
cat > /etc/wpa_supplicant/wpa_supplicant-wlan0.conf <<EOF
country=TR
ctrl_interface=DIR=/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="changeme"
    psk="changeme"
}
EOF

chmod 600 /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
systemctl disable wpa_supplicant.service
systemctl enable wpa_supplicant@wlan0.service
rfkill unblock 0
```



Servisi düzenlememiz gerekiyor ip dağıtma işlemi için ```sudo systemctl edit wpa_supplicant@wlan0.service``` varsa içindeki her şeyi silip aşağıdakini ekleyin.

```
[Service]
ExecStartPre=/sbin/ip link set %i promisc on
ExecStopPost=/sbin/ip link set %i promisc off
```

Gelelim ip dağıtma işlemine kablolu için ayrı wifi için ayrı yapılandırma aşağıdaki gibi olmalı

```
cat > /etc/systemd/network/04-wired.network <<EOF
[Match]
Name=e*

[Network]
Address=192.168.88.253/30
DHCPServer=yes

[DHCPServer]
DNS=1.1.1.1 1.0.0.1
EOF

cat > /etc/systemd/network/08-wifi.network <<EOF
[Match]
Name=wl*

[Network]
DHCP=yes
IPForward=ipv4
IPv4ProxyARP=yes
EOF
```


## Köprüleme işleminin yapılması

Kuracağımız paketlerimiz
```
apt update && apt install -y parprouted dhcp-helper
```

Kullanacağınız ```bridge.sh``` dosyamız.


```
#!/usr/bin/env bash

set -e

[ $EUID -ne 0 ] && echo "run as root" >&2 && exit 1

systemctl stop dhcp-helper
systemctl enable dhcp-helper

# Enable ipv4 forwarding.
sed -i'' s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/ /etc/sysctl.conf

# Service configuration for standard WiFi connection. Connectivity will
# be lost if the username and password are incorrect.
systemctl restart wpa_supplicant.service

# Enable IP forwarding for wlan0 if it's not already enabled.
grep '^option ip-forwarding 1$' /etc/dhcpcd.conf || printf "option ip-forwarding 1\n" >> /etc/dhcpcd.conf

# Disable dhcpcd control of eth0.
grep '^denyinterfaces eth0$' /etc/dhcpcd.conf || printf "denyinterfaces eth0\n" >> /etc/dhcpcd.conf

# Configure dhcp-helper.
cat > /etc/default/dhcp-helper <<EOF
DHCPHELPER_OPTS="-b wlan0"
EOF

# Enable avahi reflector if it's not already enabled.
sed -i'' 's/#enable-reflector=no/enable-reflector=yes/' /etc/avahi/avahi-daemon.conf
grep '^enable-reflector=yes$' /etc/avahi/avahi-daemon.conf || {
  printf "something went wrong...\n\n"
  printf "Manually set 'enable-reflector=yes in /etc/avahi/avahi-daemon.conf'\n"
}

# I have to admit, I do not understand ARP and IP forwarding enough to explain
# exactly what is happening here. I am building off the work of others. In short
# this is a service to forward traffic from WiFi to Ethernet.
cat <<'EOF' >/usr/lib/systemd/system/parprouted.service
[Unit]
Description=proxy arp routing service
Documentation=https://raspberrypi.stackexchange.com/q/88954/79866
Requires=sys-subsystem-net-devices-wlan0.device dhcpcd.service
After=sys-subsystem-net-devices-wlan0.device dhcpcd.service

[Service]
Type=forking
# Restart until wlan0 gained carrier
Restart=on-failure
RestartSec=5
TimeoutStartSec=30
# clone the dhcp-allocated IP to eth0 so dhcp-helper will relay for the correct subnet
ExecStartPre=/bin/bash -c '/sbin/ip addr add $(/sbin/ip -4 -br addr show wlan0 | /bin/grep -Po "\\d+\\.\\d+\\.\\d+\\.\\d+")/32 dev eth0'
ExecStartPre=/sbin/ip link set dev eth0 up
ExecStartPre=/sbin/ip link set wlan0 promisc on
ExecStart=-/usr/sbin/parprouted eth0 wlan0
ExecStopPost=/sbin/ip link set wlan0 promisc off
ExecStopPost=/sbin/ip link set dev eth0 down
ExecStopPost=/bin/bash -c '/sbin/ip addr del $(/sbin/ip -4 -br addr show wlan0 | /bin/grep -Po "\\d+\\.\\d+\\.\\d+\\.\\d+")/32 dev eth0'

[Install]
WantedBy=wpa_supplicant.service
EOF

systemctl daemon-reload
systemctl enable parprouted
systemctl start parprouted dhcp-helper

```

Ardından dosyası çalıştırın.


```
sudo chmod +x bridge.sh
sudo ./bridge.sh
```

Ardından isteğe bağlı olarak raspberrypi cihazınızı yeniden başlatın. Artık Raspberry Pi’de yer alan ethernet portuna cihazınızı bağlayın direk olarak ip adresi alıp internete çıkması lazım. Olmuyorsa yanlış veya bir yerlerde eksiklik vardır.

