# ALTDemo2024
# Задание 1

**Выполните базовую настройку всех устройств:**

**a. Соберите топологию согласно рисунку. Все устройства работают на OC Linux – ALT**

   **· ISP - Альт Сервер 10.2 (CLI)**

   **· CLI - Альт Рабочая станция 10.2 (GUI)**

   **· HQ-R - Альт Сервер 10.2 (CLI)**

   **· HQ-SRV - Альт Сервер 10.2 (GUI)**

   **· BR-R - Альт Сервер 10.2 (CLI)**

   **· BR-SRV - Альт Сервер 10.2 (CLI)**
**b. Присвоить имена в соответствии с топологией**

**c. Рассчитайте IP-адресацию IPv4. Необходимо заполнить таблицу №1. При необходимости отредактируйте таблицу.**

**d. Пул адресов для сети офиса BRANCH - не более 16. Для IPv6 пропустите этот пункт.**

**e. Пул адресов для сети офиса HQ - не более 64.** 

# Таблица
|Имя устройства  |Интерфейс           |IPv4            |Маска/Префикс   |Шлюз            |
|  ------------- | -------------      | -------------  |  ------------- |  ------------- |
|ISP             |ens160              |10.12.31.9      |/24             |10.12.31.254    |
|                |ens192              |DHCP            |                |                |
|                |ens224              |DHCP            |                |                |
|HQ-R            |ens160              |10.10.20.1      |/24             |                |
|                |ens192              |192.168.0.1     |/25             |                |
|BR-R            |ens160              |10.10.30.1      |/24             |                |
|                |ens192              |192.168.0.129   |/27             |                |
|HQ-SRV          |ens160              |192.168.0.2     |/25             |192.168.0.1     |
|BR-SRV          |ens160              |192.168.0.156   |/27             |192.168.0.129   |
|CLI             |ens192              |10.10.10.1      |/24             |                

# Топология
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/940d9f3c-a6a0-444f-8fea-73fcf04cb8ce)

# Сначала присвоим имена машинам
```
hostnamectl set-hostname "ИМЯ ВИРТУАЛЬНОЙ МАШИНЫ"
```
# Настройка IP-адресов
Смотрим название адаптеров
```
ip a
```
Присваиваем IP-адреса
Пример: HQ-R-SRV
```
echo 192.168.0.2/25 > /etc/net/ifaces/ens160/ipv4address
```
Шлюз
```
echo default via 192.168.0.1 > /etc/net/ifaces/ens160/ipv4route
```
Далее мы смотрим параметры интерфейсов
```
nano /etc/net/ifaces/ens160/options
```
```
BOOTPROTO=static
TYPE=eth
NM_CONTROLLED=no
DISABLED=no
CONFIG_IPV4=yes
CONFIG_IPV6=yes
```
После чего мы перезагружаем адаптер
```
systemctl restart network
```
Далее я создаю туннели "HQ-R"
```
mkdir /etc/net/ifaces/tun1
```
Создаю файлы: ipv4address, ipv4route и options
```
echo 172.168.0.1/30 > /etc/net/ifaces/tun1/ipv4address
```
```
echo default via 172.168.0.254 > /etc/net/ifaces/tun1/ipv4route
```
Захожу в options и прописываю
```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=10.10.20.1
TUNREMOTE=10.10.30.1
TUNOPTIONS='ttl 64'
HOST=ens160
```
Повторяем те же действия с BR-R.
**Данные действия мы повторяем с другими виртуальными машинами.**

# Задание 2
**Настройте внутреннюю динамическую маршрутизацию по средствам FRR. Выберите и обоснуйте выбор протокола динамической маршрутизации из расчёта, что в дальнейшем сеть будет масштабироваться.**
**Пример: HQ-R.**

Запускаем апдейт
```
apt-get update
```
Далее устанавливем утилиту FRR
```
apt-get install frr
```
Заходим в конфигурационный файл и включаем OSPF
```
nano /etc/frr/daemons
```
```
ospfd=yes
```
```
systemctl restart frr
```
Заходим в конфигурацию роутера и прописываем
```
vtysh
HQ-R# conf t
HQ-R(config)# router ospf
HQ-R(config-router)# passive-interface default 
HQ-R(config-router)# network 10.10.30.0/24 area 0
HQ-R(config-router)# network 10.10.0.0/24 area 0
HQ-R(config-router)# network 172.16.0.0/30 area 0
HQ-R(config-router)# exit
HQ-R(config)# int tun1 
HQ-R(config-if)# ip ospf network point-to-point 
HQ-R(config-if)# no ip ospf passive 
HQ-R(config-if)# do wr
```
И проверяем соседей
```
HQ-R# sh ip ospf neighbor
```
Прописываем ping и проверяем

![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/0575ab2c-a925-469d-9258-23af6009dd5a)

![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/f9dd6971-aabc-4c2e-98f5-4859461d9b91)

# Задание 3. Настройка автоматического распределения IP-адресов на роутере HQ-R.

Устанавливаем пакет
```
apt-get install dhcp-server
```
Открываем конфингурационный файл и прописывем интерфейс
```
nano /etc/sysconfig/dhcpd
```
```
DHCPDARGS=ens192
```

![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/27d8c925-a36c-4154-b33b-f7e7b2df115e)


Далее я копирую образец
```
cp /etc/dhcp/dhcpd.conf.sample /etc/dhcp/dhcpd.conf
```
Далее редактируем
```
nano /etc/dhcp/dhcpd.conf
```
И меняем
```
# See dhcpd.conf(5) for further configuration

ddns-update-style none;

subnet 192.168.0.0 netmask 255.255.255.128 {
        option routers                  192.168.0.1;
        option subnet-mask              255.255.255.128;

#       option nis-domain               "domain.org";
#       option domain-name              "domain.org";
#       option domain-name-servers      192.168.1.1;

        range dynamic-bootp 192.168.0.2 192.168.0.100;
        default-lease-time 21600;
        max-lease-time 43200;

        host HQ-SRV
        {
        hardware ethernet <MAC АДРЕС HQ-SRV!!!>;
        fixed-address 192.168.0.2;
        }
}
```
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/bd787250-79bd-490c-ab84-6187dc4ad42a)

Включаем DHCP-сервер и проверяем его
```
systemctl enable --now dhcpd
```
```
systemctl status dhcpd
```
Далее мы заходим в HQ-SRV и стираем статическую конфигурацию
```
rm -f /etc/net/ifaces/ens192/ipv4address 
rm -f /etc/net/ifaces/ens192/ipv4route
```
Заходим в options и меняем опцию получения адреса
```
BOOTPROTO=static
```
Перезагружаем адаптер и проверяем получение IP-адреса
```
systemctl restart network
```
```
ip a
```

![шзшзшзшз](https://github.com/ivantuman/ALTDemo2024/assets/148867523/611ef80e-7830-4e22-9b47-09331fafdade)


# Задание 4. Настройка локальных учётных записей на всех устройствах в соответствии с таблицей.

# Таблица
|Учётная запись  |Пароль              |Примечания          |
|  ------------- | -------------      | -------------      |  
|Admin           |P@ssw0rd            |CLI, HQ-SRV, HQ-R   |
|Branch admin    |P@ssw0rd            |BR-SRV, BR-R        |
|Network admin   |P@ssw0rd            |HQ-R, BR-R, HQ-SRV  |

Создаем пользователя admin на HQ-R
```
adduser admin
```
```
usermod -aG wheel admin
```
Задаем пароль
```
passwd admin
P@ssw0rd
P@ssw0rd
```
На CLI
```
useradd admin
```
**Делаем так же с другими виртуальными машинами**

# Задание 5. Измерение пропускной способности сети между двумя узлами HQ-R-ISP по средствам утилиты iperf 3. 

Устанавливаем утилиту iperf3 на HQ-R
```
apt-get install iperf3
```
Открываем порт
```
iptables -A INPUT -p tcp --dport 5201 -j ACCEPT
```
Прописываем адрес ISP
```
iperf3 -c 10.12.31.9
```
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/9b4418fc-a38a-4961-9c30-34153636b818)

# Задание 6. Составление backup скриптов для сохранения конфигурации сетевых устройств, а именно HQ-R BR-R. 

# 1.

Пишем простой bash-скрипт
```
nano backup-script.sh
```
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/596b35da-2225-404a-be21-09175ebf1f54)

Назначаем права на исполнение файла
```
chmod +x backup-script.sh
```
После чего проверяем работу скрипта
```
./backup-script.sh
```
И наблюдаем такую картину
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/9d1ce38c-e40c-47c4-9241-9465095a6586)
Скрипт работает

# 2.

```
mkdir /var/backup
mkdir /var/backup-script
```
```
nano /var/backup-script/backup.sh
```
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/b92cd540-a909-4dbc-be46-78380fcc600f)
```
chmod +x /var/backup-script/backup.sh
```
```
/var/backup-script/backup.sh
```

# Задание 7. Настройка подключения по SSH для удалённого конфигурирования устройства HQ-SRV по порту 2222. 

Устанавливаем пакет SSH
```
apt-get install openssh-server
```
Включаем SSH по умолчанию
```
systemctl enable --now sshd
```
Меняем порт на 2222
```
Port 2222
PermitRootLogin no
PasswordAuthentication yes
```
И перезапускаем SSH
```
systemctl restart sshd
```
Подключаемся по порту 2222
```
ssh admin@192.168.0.2 -p 2222
```
![image](https://github.com/ivantuman/ALTDemo2024/assets/148867523/692e0e20-3b58-4e37-b77e-9094020849c8)

# Задание 8. Настройка контроля доступа до HQ-SRV по SSH со всех устройств, кроме CLI.

Заходим в конфигурационный файл
```
nano /etc/openssh/sshd_config
```
И прописываю пользователей
```
AllowUsers student@192.168.0.1 student@192.168.0.158 student@192.168.0.129 student@10.12.31.9
```
