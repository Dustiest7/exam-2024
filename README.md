# **Демонстрационный экзамен.**
> [!IMPORTANT]
> **Здесь будут собраны заметки и конфигурационные файлы по дем. экзамену с учётом специфики моей местной площадки проведения.**

> [!NOTE]
>  **Задания выполнены на Alt Linux Server 10.1 и RedOS 7**

___
> **Ссылки на пройденый дем. экзамен 2024 года, по Сетевому администрированию с подробным описанием на Alt Linux Server 10.1.**

***Ссылки:***

- [Sysahelper](https://sysahelper.ru/course/view.php?id=10)

- [Backup Sysahelper](https://maxhorn20.github.io/demka/)

- [Drawio](https://www.drawio.com/)

- [Старые скрипты](https://github.com/sysadmi8812/exa2k24/tree/main/scripts)

___

# **Первый модуль. Базовая настройка устройств.**

> Выполнение работ по проектированию сетевой инфаструктуры.
## **IP-адресация на ISP**
> [!NOTE]
> **ISP является преднастроенной ВМ и организовывает доступ в сеть Интернет.**

Интерфейс | IPv4 - Адрес | IPv6 - Адрес | 
:---:       | :---:        | :---:        | 
**ens33**</br> (***ISP-HQ***)  | 11.11.11.1/24 | 2001:11::1/64 |
**ens35**</br> (***ISP-BR***)  | 22.22.22.1/24 | 2001:22::1/64 | 
**ens36**</br> (***ISP-CLI***)  | 33.33.33.1/24 | 2001:33::1/64 | 

___

## **IP-адресация на HQ-R**

Интерфейс | IPv4 - Адрес | IPv6 - Адрес | IPv4 - Шлюз | IPv6 - Шлюз | DNS
:---:       | :---:        | :---:        | :---:       | :---:       | :---:
**ens33**</br> (***ISP-HQ***)  | 11.11.11.11/24 | 2001:11::11/64 | 11.11.11.1 | 2001:11::11/64 | 11.11.11.1<br/> 8.8.8.8
**ens35**</br> (***HQ***)  | 192.168.100.62/26 | 2000:100::3f/122 | 
**ens36**</br> (***CLI-HQ***)  | 44.44.44.44/24 | 2001:44::44/64 | 


### **Настройка сетевых интерфейсов на HQ-R**

> [!NOTE]
> **На HQ-R мы реализуем статическую маршрутизацию.**

**Cоздаём для каждого интерфейса директорию по пути `/etc/net/ifaces/`:**

```
 mkdir /etc/net/ifaces/ens3{5,6}
```
**Включаем IPv6**

```
 sed -i 's/CONFIG_IPV6=no/CONFIG_IPV6=yes/g' /etc/net/ifaces/default/options
```

**Дописываем в файл `options` интерфейса `ens33`**

```
 CONFIG_IPV6=yes
```
**Копируем файл `options` для остальных двух интерфейсов**

```
 cp /etc/net/ifaces/ens33/options /etc/net/ifaces/ens35/
 cp /etc/net/ifaces/ens33/options /etc/net/ifaces/ens36/ 
```
**Назначаем IPv4 и IPv6 адреса**



**Интерфейс ens33**
```
 echo 11.11.11.11/24 > /etc/net/ifaces/ens33/ipv4address
 echo 2001:11::11/64 > /etc/net/ifaces/ens33/ipv6address
 echo default via 11.11.11.1 > /etc/net/ifaces/ens33/ipv4route
 echo default via 2001:11::1 > /etc/net/ifaces/ens33/ipv6route
```
`vim /etc/net/ifaces/ens33/resolv.conf`
```
nameserver 11.11.11.1
nameserver 8.8.8.8
```

**Интерфейс ens35**

```
 echo 192.168.100.62/26 > /etc/net/ifaces/ens35/ipv4address
 echo 2000:100::3f/122 > /etc/net/ifaces/ens35/ipv6address
```
**Интерфейс ens36**
```
 echo 44.44.44.44/24 > /etc/net/ifaces/ens36/ipv4address
 echo 2001:44::44/64 > /etc/net/ifaces/ens36/ipv6address
```
> [!IMPORTANT]
Также, поскольку HQ-R является маршрутизатором для офиса HQ - необходимо включить `forwarding` как для IPv4 так и для IPv6 - пакетов:

`vim /etc/net/sysctl.conf`

**Прописываем следующие строки:**

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```
**Для применения всех сетевых настроек перезагружаем службу `network`**

```
 systemctl restart network
```
___

## **IP-адресация на HQ-SRV**

Интерфейс | IPv4 - Адрес | IPv6 - Адрес | IPv4 - Шлюз | IPv6 - Шлюз | DNS
:---:       | :---:        | :---:        | :---:       | :---:       | :---:
**ens33**</br> (***HQ***)  | 192.168.100.1/26 | 2000:100::1/122 | 192.168.100.62 | 2000:100::3f/122 | 11.11.11.11

### **Настройка сетевых интерфейсов на HQ-SRV**

> [!NOTE]
> **На HQ-SRV первоначально мы реализуем статическую маршрутизацию, позже мы настроем DHCP-сервер, для HQ-SRV, на HQ-R.**

**Cоздаём для интерфейса директорию по пути `/etc/net/ifaces/`:**

```
 mkdir /etc/net/ifaces/ens35
```
**Включаем IPv6**

```
 sed -i 's/CONFIG_IPV6=no/CONFIG_IPV6=yes/g' /etc/net/ifaces/default/options
```

**Дописываем в файл `options` интерфейса `ens33`**

```
 CONFIG_IPV6=yes
```

**Назначаем IPv4 и IPv6 адреса**

**Интерфейс ens33**
```
 echo 192.168.100.1/26 > /etc/net/ifaces/ens33/ipv4address
 echo 2000:100::1/122 > /etc/net/ifaces/ens33/ipv6address
 echo default via 192.168.100.62 > /etc/net/ifaces/ens33/ipv4route
 echo default 2000:100::3f/122 > /etc/net/ifaces/ens33/ipv6route
```
`vim /etc/net/ifaces/ens33/resolv.conf`
```
nameserver 11.11.11.11
```
**Устанавливаем утилиту `nmtui`**
```
 apt-get update && apt-get install -y NetworkManager-{daemon,tui}
```
**Запускаем и добавляем в автозагрузку службу `NetworkManager`:**

```
 systemctl enable --now NetworkManager
```
**Для того, чтобы интерфейсы стали видимы в `nmtui` или `nmcli` - необходимо поправить параметр в файле `/etc/net/ifaces/ИМЯ_ИНТЕРФЕЙСА/options`**

```
 sed -i "s/NM_CONTROLLED=no/NM_CONTROLLED=yes/g" /etc/net/ifaces/ens33/options
```
**Перезапускаем службы `network` и `NetworkManager`:**
```
 systemctl restart network
```
```
 systemctl restart systemctl restart NetworkManager
```
___

## **IP-адресация на BR-R**

Интерфейс | IPv4 - Адрес | IPv6 - Адрес | IPv4 - Шлюз | IPv6 - Шлюз | DNS
:---:       | :---:        | :---:        | :---:       | :---:       | :---:
**ens33**</br> (***ISP-BR***)  | 22.22.22.22/24 | 2001:22::22/64 | 22.22.22.1 | 2001:22::1/64 | 22.22.22.1<br/> 8.8.8.8
**ens35**</br> (***BR***)  | 192.168.200.14/28 | 2000:200::f/124 |  | 

> [!IMPORTANT]
Также, поскольку BR-R является маршрутизатором для офиса Branch - необходимо включить `forwarding` как для IPv4 так и для IPv6 - пакетов:

**`vim /etc/sysctl.conf`**

**Прописываем следующие строки:**

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```
**Для применения всех сетевых настроек перезагружаем службу `NetworkManager`**

```
 systemctl restart NetworkManager
```

### **Настройка сетевых интерфейсов на BR-R**

> [!NOTE]
> **На BR-R мы реализуем статическую маршрутизацию.**

**Через утилиту `nmtui` настраиваем ip-адреса**
- **Имя устройства `ens33`**
- **Устройство `ens33`**<br/>
    **Конфигурация IPv4 - Вручную**<br/>
     - **Адреса `22.22.22.22/24`**
     - **Шлюз `22.22.22.1`**
     - **DNS `22.22.22.1`**<br/>
   **Конфигурация IPv6 - Вручную**<br/>
     - **Адреса `2001:22::22/64`**
     - **Шлюз `2001:22::1`**
 
- **Имя устройства `ens35`**
- **Устройство `ens35`**<br/>
    **Конфигурация IPv4 - Вручную**<br/>
     - **Адреса `192.168.200.14/28`**<br/>
   **Конфигурация IPv6 - Вручную**<br/>
     - **Адреса `2000:200::ff/124`**<br/>
     
**Включаем IPv6 через файл `/etc/sysctl.conf`**

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
```
**Затем нужно запустить `sysctl -p`, чтобы отразить изменения в системе.**
___

## **IP-адресация на BR-SRV**

Интерфейс | IPv4 - Адрес | IPv6 - Адрес | IPv4 - Шлюз | IPv6 - Шлюз | DNS
:---:       | :---:        | :---:        | :---:       | :---:       | :---:
**ens33**</br> (***BR***)  | 192.168.200.1/28 | 2000:200::1/124 | 192.168.200.14 | 2000:200::f | 22.22.22.1

### **Настройка сетевых интерфейсов на BR-SRV**

> [!NOTE]
> **На BR-SRV мы реализуем статическую маршрутизацию.**

**Через утилиту `nmtui` настраиваем ip-адреса**
- **Имя устройства `ens33`**
- **Устройство `ens33`**<br/>
    **Конфигурация IPv4 - Вручную**
     - **Адреса `192.168.200.1/28`**
     - **Шлюз `192.168.200.14`**
     - **DNS `22.22.22.1`**<br/> 
    **Конфигурация IPv6 - Вручную**<br/>
     - **Адреса `2000:200::1/124`**
     - **Шлюз `2000:200::f`**
 
**Включаем IPv6 через файл `/etc/sysctl.conf`**

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
```
**Затем нужно запустить `sysctl -p`, чтобы отразить изменения в системе.**
___

## **IP-адресация на CLI**

Интерфейс | IPv4 - Адрес | IPv6 - Адрес | IPv4 - Шлюз | IPv6 - Шлюз | DNS
:---:       | :---:        | :---:        | :---:       | :---:       | :---:
**ens33**</br> (***ISP-CLI***)  | 33.33.33.33/24 | 2001:33::33/64 | 33.33.33.1 | 2001:33::1/64 | 33.33.33.1

___

# **Первый модуль. GRE-туннель между HQ-R и BR-R**
## **Туннель на HQ-R**
> [!NOTE]
> **Forwarding IPv4 и IPv6 был включён в базовой настройке.**

**Необходимо создать директорию для туннельного интерфейса:**

**`mkdir /etc/net/ifaces/tun1`**

**Затем описываем файл options**

**`vim /etc/net/ifaces/tun1/options`**

```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=11.11.11.11
TUNREMOTE=22.22.22.22
TUNOPTIONS=`ttl 64`
HOST=ens33
```

**Назначаем IPv4 и IPv6 адрес на туннельный интерфейс:**

```
 echo 172.16.100.1/24 > /etc/net/ifaces/tun1/ipv4address
```

```
 echo 2001:100::1/64 > /etc/net/ifaces/tun1/ipv6address
```

**Перезапускаем службу `network`:**

```
systemctl restart network
```

**Включаем модуль ядра для gre:**

```
modprobe gre

```

> [!WARNING]
> **Если `modprode gre` не работает, то прописываем:**
```
/sbin/modprobe gre

```

> [!TIP]
> **Если пинги не проходят по туннелю, то можно прописать дополнительный маршрут в `ipv4route`, интерфейса `tun1`**
```
192.168.200.0/24 via 172.16.100.1
```
___

## **Туннель на BR-R**

**Через утилиту `nmtui` создаём ip-туннель**
- **Имя устройства `tun1`**
- **Устройство `tun1`**
**IP-туннель**
  - **Режим `GRE`**
  - **Родительский интерфейс `ens33`**
  -  **Локальный IP `22.22.22.22`**
  -  **Удалённый IP `11.11.11.11`**<br/>
    **Конфигурация IPv4 - Вручную**
     - **Адреса `172.16.100.2/24`**
     - **Шлюз `172.16.100.1`**
    **Конфигурация IPv6 - Вручную**
     - **Адреса `2001:100::2/64`** 

**Для корректной работы протокола динамической маршрутизации требуется увеличить параметр `TTL` на интерфейсе туннеля:**
```
 nmcli connection modify tun1 ip-tunnel.ttl 64
```       
> [!TIP]
> **Также как и на HQ-R, нам может понадобится дополнительный маршрут в интерфейсе `tun1`**

**Маршрутизация**
 - **Назначение/Префикс `192.168.100.0/24`**
 - **Следующий переход `172.16.100.1`**
 - **Метрика `1`**
___

# **Первый модуль. Настройка DHCP-Сервера на HQ-R.**
## **Настройка DHCP-сервера для IPv4**

**Установка пакета `dhcp-server`:**

```
 apt-get update && apt-get install -y dhcp-server
```

**Укажим сетевой интерефейс, через который будет работать DHCP-сервер:**

`/etc/sysconfig/dhcpd` - для dhcpd.service<br/>
`/etc/sysconfig/dhcpd6` - для dhcpd6.service

```
 DHCPDARGS=ens35
```

**Далее опишим конфигурационный файл для DHCP работающего с IPv4:**

```
# dhcpd.conf
#
default-lease-time 6000;
max-lease-time 72000;

authoritative;

subnet 192.168.100.0 netmask 255.255.255.192 {
  range 192.168.100.5 192.168.100.61;
  option routers 192.168.100.62;
  }

host hq-srv {
  hardware ethernet 00:0c:29:03:aa:ca;
  fixed-address 192.168.100.1;
  
}
```

**После чего можно проверить данный конфигурационный файл через утилиту `dhcpd`**

```
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```
> [!NOTE]
> В случае ошибки в описании конфигурационного файла - в выводе данной утилиты будет написано что не так.

**Запускаем и добавляем в автозагрузку службу `dhcpd` (для IPv4):**
```
 systemctl enable --now dhcpd
```
**Запускаем журнал и включаем или перезагружаем интерфейс на HQ-SRV:**
```
 journalctl -f -u dhcpd
```
___

## **Настройка DHCP-сервера для IPv6**

**Укажим сетевой интерефейс, через который будет работать DHCP-сервер:**

`/etc/sysconfig/dhcpd6` - для dhcpd6.service

```
 DHCPDARGS=ens35
```

**Опишим конфигурационный файл для DHCP работабщего с IPv6:**


```
# Server configuration file example for DHCPv6
 default-lease-time 2592000;
 preferred-lifetime 604800;
 option dhcp-renewal-time 36000;
 option dhcp-rebinding-time 72000;

 allow leasequery;

 option dhcp6.preference 255;

 option dhcp6.info-refresh-time 21600;

 subnet6 2000:100::/122 {
         range6 2000:100::2 2000:100::3f;
 }

host hq-srv {

	host-identifier option
		dhcp6.client-id 00:04:bc:9c:dd:4c:4b:5c:7c:2f:b2:22:27:b5:17:5e:62:de;

	fixed-address6 2000:100::1;

        fixed-prefix6 2000:100::/122;
}
```
**Запускаем и добавляем в автозагрузку службу `dhcpd6`:**
```
 systemctl enable --now dhcpd6
```
> [!WARNING]
> Если `dhcpd.service` или `dhcpd6.service` выдаёт при запуске ошибку, узнать в чём конкретно заключается проблема, можно с помошью `journalctl _PID=номер_процесса`

**После этих манипуляций HQ-SRV получит IPv4 и IPv6 адрес автоматически.**
- **Средствами `nmtui` удаляем и создаём заново интерфейс `ens33` с параметром `Automatic` и проверяем.**
- **Прописываем `ip –c a` и в описании интерфейса должна появится надпись `dynamic`, это означает что мы всё настроили правильно.**

> [!IMPORTANT]
> **Для IPv4 на HQ-SRV без проблем прилетит шлюз, т.к. за это отвечает параметр `option routers` в настройках `dhcpd.conf`**<br/>
> **Для IPv6 такого параметра нет, шлюзы IPv6 выдаются маршрутизаторами средствами `RA (Router Advertisement)`**

___

## **Установка и настройка RA:**

**Установка пакета `radvd`:**

```
 apt-get install -y radvd
```

**Следующий модуль отвечает за RA на интерфейсе `ens35` (нужно для получения HQ-SRV верных параметров IPv6 автоматически):**

```
 echo net.ipv6.conf.ens35.accept_ra = 2 >> /etc/net/sysctl.conf; systemctl restart network
```

**Файл конфигурации по умолчанию находится в `/etc/radvd.conf`**

**Опишим конфигурационный файл для RA:**

```
# NOTE: there is no such thing as a working "by-default" configuration file.
#       At least the prefix needs to be specified.  Please consult the radvd.conf(5)
#       man page and/or /usr/share/doc/radvd-*/radvd.conf.example for help.
#
#
interface ens35
{
	AdvSendAdvert on;
	AdvManagedFlag on;
	AdvOtherConfigFlag on;
	prefix 2000:100::/122
	{
		AdvOnLink on;
		AdvAutonomous on;
		AdvRouterAddr on;
	};

};
```

**Далее - необходимо перезапустить `dhcpd6.service` и запустить и добавить в автозагрузку `radvd`:**

```
 systemctl restart dhcpd6
```
```
 systemctl enable --now radvd
```
___

# **Первый модуль. Создание локальных учётных записей.**
> [!NOTE]
> **Необходимо дважды указать в качестве пароля - `P@ssw0rd`**

## **CLI**
```
useradd admin -m -c "Admin" -U
```
```
passwd admin
```

## **HQ-R**

```
useradd admin -m -c "Admin" -U
```
```
passwd admin
```
```
useradd network-admin -m -c "Network admin" -U
```
```
passwd network-admin
```
## **HQ-SRV**
```
useradd admin -m -c "Admin" -U
```
```
passwd admin
```
## **BR-R**
```
useradd branch-admin -m -c "Branch admin" -U
```
```
passwd branch-admin
```
```
useradd network-admin -m -c "Network admin" -U
```
```
passwd network-admin
```
## **BR-SRV**
```
useradd branch-admin -m -c "Branch admin" -U
```
```
passwd branch-admin
```
```
useradd network-admin -m -c "Network admin" -U
```
```
passwd network-admin
```
**Проверям создание записей с помошью команды `grep admin /etc/passwd`**
___

# **Первый модуль. Измерение пропускной способности сети между двумя узлами.**
**Измерьте пропускную способность сети между двумя узлами HQ-R-ISP по средствам утилиты `iperf3`.<br/> Предоставьте описание пропускной способности канала со скриншотами.**

**Устанавливаем пакет `iperf3`**
```
apt-get install -y iperf3
```
**Выполняем запуск службы `iperf3`**
```
systemctl enable --now iperf3
```
**Запустим `iperf3` в режиме клиента, используя флаг `-c`, и укажем хост, на котором работает сервер -> `ISP (11.11.11.1)`**

```
iperf3 -c 11.11.11.1
```

**При помощи параметра `--logfile /path/to/file` - можно указать путь к файлу в который записать вывод данной каманды в качестве отчёта**

```
iperf3 -c 11.11.11.1 --get-server-output --logfile ~/iperf3_logfile.txt
```
___

# **Первый модуль. Backup скрипты.**
**Составим простой backup скрипт для сохранения конфигурации сетевых устройств.**

**`vim backup-script.sh`**

```
#!/bin/bash

echo "Start backup!"

backup_dir="/etc"
dest_dir="/opt/backup"

mkdir -p $dest_dir
tar -czf $dest_dir/$(hostname -s)-$(date +"%d.%m.%y").tgz $backup_dir

echo "Done!"
```
**Назначаем права на исполнения для данного файла:**

`chmod +x backup-script.sh`

**Выполняем запуск скрипта:**

`./backup-script.sh`

**Просмотрим содержание архива:**

`tar -tf /opt/backup/имя_архива | less`

> [!NOTE]
> **Таким образом, скрипт записал в архив всё содерджимое директории `/etc`**

___

# **Первый модуль. Настройка подключения по SSH.**
## **HQ-SRV:**
**Меняем стандартный порт ssh (22) на 2020 согласно заданию:**

```
 sed -i "s/#Port 22/Port 2020/g" /etc/openssh/sshd_config
```

**Перезапускаем службу `sshd`:**

```
 systemctl restart sshd
```
> [!NOTE]
>**Перенаправить трафик на этот порт по средствам контролирования трафика - можно посредством настройки `firewall (netfilter)` на HQ-R**

## **HQ-R**

**Выполним установку `nftables`:**

```
 apt-get install -y nftables
```

**Включаем и добавляем в автозагрузку службу `nftables`:**

```
 systemctl enable --now nftables
```

**Создадим следующее правило, которое будет перенаправлять внешние подключения на порт 22 -> на порт 2020 сервера HQ-SRV:**

- **Cоздадим таблицу `nat` - для семейства `inet`**
  - **Семейство `inet` объединяет протоколы IPv4 и IPv6**

```
 nft add table inet nat
```

**Добавим цепочку `prerouting` в таблицу `nat`**

```
 nft add chain inet nat prerouting '{ type nat hook prerouting priority 0; }'
```

**Добавим необходимое правило для проброса портов в цепочку `prerouting` таблицы `nat` семейства `inet`:**

```
 nft add rule inet nat prerouting ip daddr 11.11.11.11 tcp dport 22 dnat to 192.168.100.1:2020
```

```
 nft add rule inet nat prerouting ip6 daddr 2001:11::11 tcp dport 22 dnat to [2000:100::1]:2020
```

**Просмотр текущих правил:**

```
 nft list ruleset | cat -n
```

**Для сохранения текущих правил, необходимо перенаправить результат вывода данной команды, а именно строки с 14 по 20 в конфигурационный файл `/etc/nftables/nftables.nft`**

```
 nft list ruleset | tail -n 7 | tee -a /etc/nftables/nftables.nft
```

**Перезапускаем службу `nftables`:**

```
 systemctl restart nftables
```
___

# **Первый модуль. Настройка контроля доступа до HQ-SRV по SSH.**
## **HQ-R:**

**Через ранее установленный nftables настраиваем доступ в Интернет (NAT), чтобы можно было устанавливать пакеты внутри локальной сети офиса HQ:**
- **В ранее созданную таблицу nat в семействе inet добавляем цепочку postrouting:**

  ```
  nft add chain inet nat postrouting '{ type nat hook postrouting priority 0; }'
  ```
**В созданную цепочку postrouting добавляем правило, реализующее nat (masquerade):**

```
nft add rule inet nat postrouting ip saddr 192.168.100.0/26 oifname 'ens33' counter masquerade
```
## **HQ-SRV:**
 **Обновляем список пакетов и устанавливаем `nftables`**

```
apt-get update && apt-get install -y nftables
```
**Включаем и добавляем в автозагрузку `nftables`**

```
systemctl enable --now nftables
```
**Добавляем правила, запрещающие доступ по `ssh` (порт 2020) с `CLI`, как с временного подключения так и с глобального, как для IPv4 так и для IPv6:**
- **для IPv4**

  ```
  nft add rule inet filter input ip saddr 33.33.33.33 tcp dport 2020 counter drop
  ```

  ```
  nft add rule inet filter input ip saddr 44.44.44.0/24 tcp dport 2020 counter drop
  ```

  - **для IPv6**
 
    ```
    nft add rule inet filter input ip6 saddr 2001:33::/64 tcp dport 2020 counter drop
    ```
    ```
    nft add rule inet filter input ip6 saddr 2001:44::/64 tcp dport 2020 counter drop
    ```

  **Смотрим правила с помошью команды `nft list ruleset`**<br/>
  **Сохраняем правила `nftables`:**
- **В конфигурационном файле `/etc/nftables/nftables.nft` - удаляем все не закомментированные записи:**

  **Отправляем результат команды `nft list ruleset` - в файл `/etc/nftables/nftables.nft`**
    ```
    nft list ruleset | tee -a /etc/nftables/nftables.nft
    ```
  **Перезапускаем `nftables` и проверяем наличие правил:**
    ```
    systemctl restart nftables
    ```

    ___

    # **Второй модуль. Организация сетевого администрирования**
    ## **Настройка доступа в Интернет из локальных сетей офисов HQ и Branch**

  **Настраиваем `NAT` - через `nftables` на `HQ-R` и `BR-R`**

  **Обновляем список пакетов и устанавливаем `nftables`**

   ```
   apt-get update && apt-get install -y nftables
   ```
  **Создаём таблицу `nat` для семейства `inet` (ipv4 и ipv6):**
  ```
  nft add table inet nat
  ```
  **Создаём цепочку `postrouting` (после маршрутизации) в таблице `nat` семейства `inet`:**

  ```
  nft add chain inet nat postrouting '{ type nat hook postrouting priority 0; }'
  ```

  **Добавляем правило в цепочку `postrouting` для `HQ-R`:**
  
  ```
  nft add rule inet nat postrouting ip saddr 192.168.100.0/26 oifname 'ens33' counter masquerade
  ```
  **Добавляем правило в цепочку `postrouting` для `BR-R`:**

  ```
  nft add rule inet nat postrouting ip saddr 192.168.200.0/28 oifname 'ens33' counter masquerade
  ```

 **На `HQ-R` Отправляем результат команды `nft list ruleset` - в файл `/etc/nftables/nftables.nft`**
 
   ```
   nft list ruleset | tee -a /etc/nftables/nftables.nft
   ```
    
  **Перезапускаем `nftables` и проверяем наличие правил:**
    ```
    systemctl restart nftables
    ```

   **На `BR-R` Отправляем результат команды `nft list ruleset` - в файл `/etc/sysconfig/nftables.conf`**
   
  ```  
  nft list ruleset > /etc/sysconfig/nftables.conf
  ``` 
    
  **Перезапускаем `nftables` и проверяем наличие правил:**
    ```
    systemctl restart nftables
    ```
___

 ## **Второй модуль. Настройка DNS-сервера на HQ-SRV**

 **Устанавливаем пакеты `bind` и `bind-utils`:**

 ```
 apt-get install -y bind bind-utils
 ```

**В конфигурационном файле `/etc/bind/options.conf` - правим следующие параметры:**

`vim /etc/bind/options.conf`

```
listen-on { any; };
listen-on-v6 {any; };

forward first;
forwarders { 8.8.8.8; };

allow-query { any; };
 ```
**Запускаем и добавляем в автозагрузку службу `bind`:**

```
systemctl enable --now bind
```

**В качестве `DNS`-сервера для `hq-srv` должен быть `127.0.0.1`:**

```
echo name_servers=127.0.0.1 >> /etc/resolvconf.conf
```
```
resolvconf -u 
```

**В конфигурационном файле `/etc/bind/local.conf` описываем необходимые зоны согласно требованию задания:**

- **`hq.work` - зона прямого просмотра;**
- **`branch.work` - зона прямого просмотра;**
- **`100.168.192.in-addr.arpa` - зона обратного просмотра;**
- **`200.168.192.in-addr.arpa` - зона обратного просмотра;**
    
  `vim /etc/bind/local.conf`

```
include "/etc/bind/rfc1912.conf";

// Consider adding the 1918 zones here,
// if they are not used in your organization.
//	include "/etc/bind/rfc1918.conf";

// Add other zones here

  zone "hq.work" {
        type master;
	file "hq.db";
  };

  zone "branch.work" {
        type master;
	file "branch.db";
  };

  zone "100.168.192.in-addr.arpa" {
        type master;
	file "100.db";
  };

  zone "200.168.192.in-addr.arpa" {
        type master;
	file "200.db";
  };

```

**Правим файл зоны прямого просмотра для `hq.work`:**

`vim /etc/bind/zone/hq.db`

```
$TTL	1D
@	IN	SOA	hq.work. root.hq.work. (
				2024021400	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	hq.work.
	IN	A	127.0.0.0
hq-r    IN      A       192.168.100.62
hq-srv  IN      A       192.168.100.1
```

**Правим файл зоны прямого просмотра для `branch.work`:**

`vim /etc/bind/zone/branch.db`

```
$TTL	1D
@	IN	SOA	branch.work. root.branch.work. (
				2024021400	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	branch.work.
	IN	A	127.0.0.0
br-r    IN      A       192.168.200.14
br-srv  IN      A       192.168.200.1
```

**Правим файл зоны обратного просмотра для `hq.work`  - `100.db`:**

`vim /etc/bind/zone/100.db`

```
$TTL	1D
@	IN	SOA	hq.work. root.hq.work. (
				2024021400	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	hq.work.
62	IN	PTR	hq-r.hq.work.
1	IN	PTR	hq-srv.hq.work.
```

**Правим файл зоны обратного просмотра для `branch.work`  - `200.db`:**

`vim /etc/bind/zone/200.db`

```
$TTL	1D
@	IN	SOA	branch.work. root.branch.work. (
				2024021400	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	branch.work.
14	IN	PTR	br-r.branch.work.
```
**Проверить файлы зон можно утилитой `named-checkconf`**

```
named-checkconf -z
```
> [!IMPORTANT]
> **Задаём необходимые права:**
>
> ```
> chown root:named /etc/bind/zone/{hq,branch,100,200}.db
> ```

**Перезапускаем службу `bind`**

```
systemctl restart bind
```

___


















