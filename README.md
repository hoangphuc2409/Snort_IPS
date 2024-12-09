# Snort_IPS
## Cấu hình router

Ens34, Ens35 là gateway cho mạng 10.81.59.0/24 và 192.168.59.0/24

Network interface Ens33 cấu hình NAT để Ens34 và Ens35 ra internet

Cài đặt các gói phụ thuộc
```
sudo apt install openvswitch-switch
sudo install iptables-persistent
```

Mở file cấu hình
```
nano /etc/netplan/50-cloud-init.yaml
```

Cấu hình IP như sau:
```
network:
   ethernets:
       ens33:
           dhcp: true
       ens34:
           dhcp: false
           addresses: [10.81.59.1/24]
           nameservers:
            addresses: [8.8.8.8]
       ens35:
           dhcp: false
           addresses: [192.168.59.1/24]
           nameservers:
            addresses: [8.8.8.8]
   version: 2
```
Lưu lại cấu hình
```
sudo netplan apply
```

**Cấu hình NAT outbound**

Xóa rule cũ
```
iptables --flush
iptables --table nat --flush
iptables --delete-chain
iptables --table nat --delete-chain
```
Cấu hình Ens34 và Ens35 sẽ đi ra internet thông qua Ens33

```
iptables --table nat --append POSTROUTING --out-interface ens33 -j MASQUERADE
iptables --append FORWARD --in-interface ens34 -j ACCEPT
iptables --append FORWARD --in-interface ens35 -j ACCEPT
echo 1 > /proc/sys/net/ipv4/ip_forward
service iptables restart
```

## Cấu hình máy Attacker
Máy attacker có IP là 10.81.59.100

Mở file cấu hình
```
sudo nano /etc/network/interfaces
```
Cấu hình static IP
```
auto eth0
iface eth0 inet static
address 10.81.59.100/24
gateway 10.81.59.1
```
Áp dụng thay đổi

```
sudo systemctl restart networking
```

Mở file cấu hình DNS

```
sudo nano /etc/resolv.conf
```

Thêm dòng sau và lưu lại file

```
nameserver 8.8.8.8
```

## Cấu hình máy Snort
Cài đặt Snort và kiểm tra version. Phiên bản trong bài sử dụng là Snort 2.9.7.0

```
sudo apt-get install snort
snort -V
```
Kiểm tra DAQ modules
```
sudo snort --daq-list
```

**Xóa rule cũ và tạo rule mới**

```
sudo rm -rf /etc/snort/rules/*
sudo nano /etc/snort/rules/nhom7.rules
sudo nano /etc/snort/nhom7.conf
```

**Tạo file cấu hình**

```
sudo nano /etc/snort/nhom7.conf
```
**Chỉnh sửa file cấu hình để bật inline mode**

```
config daq: afpacket
config daq_mode: inline
```
**Kiểm tra Snort**

``
sudo snort -T -c /etc/snort/nhom7.conf -Q -i ens34:ens35
``

**Khởi động Snort:** 

``
sudo snort -c /etc/snort/nhom7.conf -Q -i ens34:ens35
``

## Cấu hình máy victim
Máy victim có IP là 192.168.59.200
```
sudo nano /etc/network/interfaces
```

Cấu hình static IP

```
auto eth0
iface eth0 inet static
address 192.168.59.200/24
gateway 192.168.59.1
```

Áp dụng thay đổi

```
sudo systemctl restart networking
```

Mở file cấu hình DNS

```
sudo nano /etc/resolv.conf
```

Thêm dòng sau và lưu lại file

```
nameserver 8.8.8.8
```

Máy victim chỉ có thể ra được internet khi Snort được chạy.
