## Static IP change

```
└─$ nmcli connection show
NAME                UUID                                  TYPE      DEVICE 
Wired connection 1  54c4bdc0-b27c-47ca-a22d-cf8a08dc0d95  ethernet  eth0   
lo                  9529fe6e-db5c-42f2-814a-ac715d041702  loopback  lo     


┌──(lucius㉿vbox)-[~]
└─$ nmcli connection modify "Wired connection 1" \
ipv4.method manual \
ipv4.addresses 192.168.10.99/24 \
ipv4.gateway 192.168.10.1 \
ipv4.dns 8.8.8.8


┌──(lucius㉿vbox)-[~]
└─$ nmcli connection down "Wired connection 1"

Connection 'Wired connection 1' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/2)


┌──(lucius㉿vbox)-[~]
└─$ nmcli connection up "Wired connection 1"

Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)


┌──(lucius㉿vbox)-[~]
└─$ ip a    
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:47:79:15 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.99/24 brd 192.168.10.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe47:7915/64 scope link tentative noprefixroute 
       valid_lft forever preferred_lft forever

```

