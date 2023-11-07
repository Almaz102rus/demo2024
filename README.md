
#### №1.1
## Описание задания:

1. Выполните базовую настройку всех устройств:
	a. Соберите топологию согласно рисунку. Все устройства работают на OC Linux - Debian
b. Присвоить имена в соответствии с топологией
c. Рассчитайте IP-адресацию IPv4 и IPv6. Необходимо заполнить таблицу №1. При необходимости отредактируйте таблицу.
d. Пул адресов для сети офиса BRANCH - не более 16. Для IPv6 пропустите этот пункт.
e. Пул адресов для сети офиса HQ - не более 64. Для IPv6 пропустите этот пункт.

## Топология сети:
![image](https://github.com/Almaz102rus/demo2024/assets/148868440/6197b9f2-47b8-4351-8fad-7c1b5ad121d1)



 ## Таблица разбиения сети на подсети
                    
| Имя устр.    | Интерфейс |  IPv4         |  Маска/Префикс  |  Шлюз       |
| -----------  | --------- |-------------- | ----------------|-------------|
|    ISP       | ens 192   | DHCP          | 24              |             |
|              | ens 224   | 192.168.0.162 | 30              |             |
|              | ens 256   | 192.168.0.165 | 30              |             |
|    HQ-R      | ens 192   | 192.168.0.1   | 25              |             |
|              | ens 224   | 192.168.0.166 | 30              |192.168.0.165|
|    BR-R      | ens 224   | 192.168.0.161 | 30              |192.168.0.162|
|              | ens 192   | 192.168.0.129 | 27              |             |
|    HQ-SRV    | ens 192   | 192.168.0.2   | 25              |192.168.0.1  |
|    BR-SRV    | ens 192   | 192.168.0.130 | 27              |192.168.0.129|


Я ввел команду для просмотра конфигурации IP адресов: 
```
nano /etc/network/interfaces
```
Далее, добавил необходимые IP в соотвествии с таблицей.
BR-R
```
auto ens192
iface ens192 inet static
address 192.168.0.129
netmask 255.255.255.224

auto ens224
iface ens224 inet static
address 192.168.0.161
netmask 255.255.255.252
gateway 192.168.0.162
```
BR-SRV
```
auto ens192
iface ens192 inet static
address 192.168.0.130
netmask 255.255.255.224
gateway 192.168.0.129
```
HQ-R
```
auto ens192
iface ens192 inet static
address 192.168.0.1
netmask 255.255.255.128

auto ens224
iface ens224 inet static
address 192.168.0.166
netmask 255.255.255.252
gateway 192.168.0.165
```
HQ-SRV
```
auto ens192
iface ens192 inet static
address 192.168.0.2
netmask 255.255.255.128
gateway 192.168.0.1
```
ISP
```
auto ens192
iface ens192 inet dhcp

auto ens224
iface ens192 inet static
address 192.168.0.162
netmask 255.255.255.252

auto ens256
iface ens192 inet static
address 192.168.0.165
netmask 255.255.255.252
```
Сохраняю конфигурацию: CTRL+S
Выхожу из конфигурации: CTRL+X

#### №1.2
##Описание задания:
1.2 Настройте внутреннюю динамическую маршрутизацию по средствам FRR. Выберите и обоснуйте выбор протокола динамической маршрутизации из расчёта, что в дальнейшем сеть будет масштабироваться.
1.2.1 Составьте топологию сети L3.
## NAT(ISP, HQ-R, BR-R)
Делаю установку пакета FRR.
Для этого необходимо войти в конфигурацию /etc/sysctl.conf и добавить строку net.ipv4.ip_forward=1
Для применения изменений без перезагрузки использую команду:
```
sysctl -p
```
Включаю NAT командой:
```
iptables -A POSTROUTING -t nat -j MASQUERADE
```
Ввожу nato /etc/network/if-pre-up.d/ для создания файла "nat" и ввожу следующие строки:
```
#!/bin/sh
/sbin/iptables -A POSTROUTNIG -t nat -j MASQUERADE
```
Осталось сделать файл запускаемым: 
chmod +x /etc/network/if-pre-up.d/nat

## FRR OSPF(ISP, HQ-R, BR-R)
Делаю установку FRR 
```
apt update
apt install frr
```
nano /etc/frr/daemons
Нахожу строку ospfd=no и меняю его на ospfd=yes
```
systemctl restart frr
```
Для входа в среду роутера, ввожу `vtysh`
Вывести информацию об интерфейсе:
```
do sh int br
```
```
conf t
router ospf
```
```
192.168.0.0/25 area 0
192.168.0.164/30 area 0
```
Для того, чтобы узнать, в какой зоне работает интерфейс и инфу о соседях:
```
sh ip ospf neighbor
```
![image](https://github.com/Almaz102rus/demo2024/assets/148868440/671682e1-e140-474b-b623-251e2a2c33fc)

#### №1.3 DHCP Сервер на HQ-R
##Описание задания:
1.3 Настройте автоматическое распределение IP-адресов на роутере HQ-R.
1.3.1 Учтите, что у сервера должен быть зарезервирован адрес.
Установка DHCP:
```
apt update
apt install isc-dhcp-server
```
Config:
```
nano /etc/default/isc-dhcp-server
```
Указываю интерфейс, который указывает на ISP:
```
INTERFACESV4="ens192"
```
Настройка раздачи IP-адресов:
```
nano /etc/dhcp/dhpcd.conf

subnet 192.168.0.0 255.255.255.128
{range 192.168.0.2 192.168.0.125;
option domain-name-servers 8.8.8.8 8.8.4.4;
option routers 192.168.0.1;}
```
Для применения изменений:
```
systemctl restart isc-dhcp-server.service
```


