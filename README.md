# DEM24
**Ультимативная записная книжка по дем экзамену 2024**

[На всякий](https://github.com/NyashMan/DEMO2024)

## Шаги выполнения

### 1. Установить IP
### 2. Проверить пинги
### 3. Установить альтератор на ISP

```bash
apt-get install alterator-fbi
service alteratord start
service ahttpd start
```

### 4. На CLI выставить DNS

```bash
10.104.1.100
```

### 5. Выставить туннели на HQ-R и BR-R

Настроить туннели, поменяв местами 10.x.0.2 и 10.x.2.0.

### 6. Установить FRR и настроить OSPF

#### HQ-R

```bash
# Установить FRR
apt-get install frr

# Изменить конфигурацию
nano /etc/frr/daemons
# Изменить строку:
ospfd=no
# на:
ospfd=yes

systemctl enable --now frr

# Конфигурация OSPF
vtysh
conf t
router ospf
passive-interface default
network 10.0.0.0/26 area 0
network 172.16.0.0/24 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do write memory
exit
```

```bash
# Задать TTL для OSPF
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
```

```bash
# Отключить Firewalld
systemctl stop firewalld.service
systemctl disable --now firewalld.service
systemctl restart frr
```

#### BR-R

```bash
# Установить FRR
apt-get install frr

# Изменить конфигурацию
nano /etc/frr/daemons
# Изменить строку:
ospfd=no
# на:
ospfd=yes

systemctl enable --now frr

# Конфигурация OSPF
vtysh
conf t
router ospf
passive-interface default
network 10.0.2.0/28 area 0
network 172.16.0.0/24 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do write memory
exit
```

```bash
# Задать TTL для OSPF
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
```

```bash
# Отключить Firewalld
systemctl stop firewalld.service
systemctl disable --now firewalld.service
systemctl restart frr
```

### Проверка работы OSPF

```bash
vtysh
show ip ospf neighbor
exit
```

Если OSPF не заработал, перезапустить машины ISP, BR-R и HQ-R.

### 7. Настройка автоматического распределения IP-адресов на роутере HQ-R

#### HQ-R

```
# Настройка DHCP сервера
nano /etc/sysconfig/dhcpd
# Добавить:
DHCPDARGS=ens224

cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
```
```
#dhcpd.conf
option domain-name "HQ-R";
option domain-name-servers 10.0.0.1;

default-lease-time 6000;
max-lease-time 72000;

authoritative;

subnet 10.0.0.0 netmask 255.255.255.192 {
  range 10.0.0.1 10.0.0.62;
  option routers 10.0.0.1;
}

host HQ-SRV {
  hardware ethernet 00:50:56:A8:6E:43;
  fixed-address 10.0.0.2;
}
```

# Проверка файла и запуск службы
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl enable --now dhcpd
systemctl status dhcpd
journalctl -f -u dhcpd
```

### 8. Настройка локальных учётных записей

#### Таблица 2
```

| Логин         | Пароль    | На серверах           |
|---------------|-----------|-----------------------|
| Admin         | P@ssw0rd  | CLI, HQ-SRV, HQ-R     |
| Branch admin  | P@ssw0rd  | BR-SRV, BR-R          |
| Network admin | P@ssw0rd  | HQ-R, BR-R, BR-SRV    |
```
```
#### Команды для создания учетных записей

На CLI:


Система -> Центр управления системой -> Создать учетную запись (тип: Admin, входит в группу администраторов, пароль)


На BR-R, BR-SRV, HQ-SRV:

```
useradd branch-admin -m -c "Branch admin" -U && passwd branch-admin
useradd network-admin -m -c "Network admin" -U && passwd network-admin
```

На HQ-R:

```
useradd network-admin -m -c "Network admin" -U && passwd network-admin
useradd admin -m -c "Admin" -U && passwd admin
```


### 9. Измерение пропускной способности сети между HQ-R и ISP с помощью iperf3

#### HQ-R

```
apt-get -y install iperf3
systemctl enable --now iperf3
```

#### Тестирование на HQ-R
(за место того нужно вставить HQ-R который идёт на ISP)
По результату проверки сделать скриншот)
```
iperf3 -c <ISP_IP> -f m --get-server-output
```

Замените `<ISP_IP>` на IP-адрес ISP.
