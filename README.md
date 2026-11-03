# Лабораторна робота №4 - Служби DNS та NAT. Підходи до траблшутінгу мережі

## Мета

Ознайомитися з технологією NAT. Налаштувати NAT за допомогою iptables. Розібратися з підходами до траблшутингу мережі.

## Середовище

Роботу виконано на одному ноутбуці (Arch Linux). Другий комп'ютер (PC2) змодельовано мережевим простором імен `pc2`, з'єднаним з основною системою (PC1) парою `veth`.

| Вузол / інтерфейс             | Роль                                 | IPv4             | MAC               |
| ----------------------------- | ------------------------------------ | ---------------- | ----------------- |
| PC1, `wlan0`                  | Зовнішній інтерфейс NAT              | 192.168.50.32/24 | 14:ac:60:6e:b9:f1 |
| PC1, `veth-pc1`               | Шлюз внутрішньої мережі, DHCP-сервер | 10.0.15.1/24     | f2:da:32:a2:72:fe |
| PC2 (netns `pc2`), `veth-pc2` | Клієнт, адреса через DHCP            | 10.0.15.10/24    | fa:06:cd:5f:6c:99 |
| Роутер ASUS                   | Шлюз домашньої мережі, DNS           | 192.168.50.1     | bc:fc:e7:ac:8f:f0 |

ПЗ: iptables (nf_tables), ISC DHCP (`dhcp`, `dhclient`), conntrack-tools, Wireshark.

## Конфігурація

`/etc/dhcpd.conf` (обслуговує лише внутрішню мережу, для `192.168.50.0/24` оголошення `subnet` немає):

```conf
default-lease-time 600;
max-lease-time 7200;
option subnet-mask 255.255.255.0;
option broadcast-address 10.0.15.255;
option routers 10.0.15.1;
option domain-name-servers 192.168.50.1, 8.8.8.8;
option domain-name "mydomain.example";

subnet 10.0.15.0 netmask 255.255.255.0 {
    range 10.0.15.10 10.0.15.20;
}
```

NAT і маршрутизація:

```bash
# /etc/sysctl.d/30-ipforward.conf
net.ipv4.ip_forward = 1

# правило NAT (збережено в /etc/iptables/iptables.rules)
-t nat -A POSTROUTING -s 10.0.15.0/24 -o wlan0 -j MASQUERADE
```

## Відтворення

```bash
# 1. Пакети
sudo pacman -S --needed dhcp dhclient conntrack-tools net-tools wireshark-qt

# 2. PC2 у network namespace
sudo ip netns add pc2
sudo ip link add veth-pc1 type veth peer name veth-pc2
sudo ip link set veth-pc2 netns pc2
sudo ip addr add 10.0.15.1/24 dev veth-pc1
sudo ip link set veth-pc1 up
sudo ip netns exec pc2 ip link set lo up
sudo ip netns exec pc2 ip link set veth-pc2 up

# 3. DHCP-сервер (конфіг вище) і клієнт у pc2
sudo dhcpd -t -cf /etc/dhcpd.conf
sudo systemctl restart dhcpd4
sudo mkdir -p /etc/netns/pc2
echo "nameserver 192.168.50.1" | sudo tee /etc/netns/pc2/resolv.conf
sudo ip netns exec pc2 dhclient -v -pf /run/dhclient-pc2.pid \
     -lf /var/lib/dhclient/dhclient-pc2.leases veth-pc2

# 4. Маршрутизація та NAT
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/30-ipforward.conf
sudo iptables -t nat -A POSTROUTING -o wlan0 -s 10.0.15.0/24 -j MASQUERADE
sudo iptables-save -f /etc/iptables/iptables.rules
sudo systemctl enable iptables.service

# 5. Перевірка (захоплення у Wireshark на veth-pc1 + wlan0)
sudo ip netns exec pc2 ping -c 5 google.com          # ping-streaming.pcapng
sudo conntrack -L -p icmp

# 6. Траблшутинг (захоплення на veth-pc1)
sudo ip neigh flush dev veth-pc1
ping -c 4 10.0.15.10                                 # local-connection-troubleshooting.pcapng
sudo ip netns exec pc2 ping -c 4 10.0.15.1
sudo ip netns exec pc2 ping -c 4 192.168.50.1        # pc2-dns-troubleshooting.pcapng
sudo ip netns exec pc2 ping -c 4 8.8.8.8

# 7. Відтворення дефекту (захоплення на veth-pc1 + wlan0)
sudo mv /etc/sysctl.d/30-ipforward.conf /root/
echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -w net.ipv4.ip_forward=0                 # стан після перезавантаження
sudo systemctl restart systemd-sysctl
sysctl net.ipv4.ip_forward                           # => 0
sudo ip netns exec pc2 ping -c 4 10.0.15.1           # defect-localization.pcapng
sudo ip netns exec pc2 ping -c 4 8.8.8.8

# 8. Виправлення
sudo mv /root/30-ipforward.conf /etc/sysctl.d/
sudo sed -i "/net.ipv4.conf.all.forwarding=1/d" /etc/sysctl.conf
sudo systemctl restart systemd-sysctl                # => ip_forward = 1

```

## Файли захоплення

Щоб у файлах з двома інтерфейсами бачити, на якому з них захоплено пакет, додайте у Wireshark стовпець `frame.interface_name`
(_Edit => Preferences => Columns_ або ПКМ по полі _Interface name_ у деталях кадру => _Apply as Column_).

| Файл                                      | Інтерфейси          | Фільтр          | Що показує                                            |
| ----------------------------------------- | ------------------- | --------------- | ----------------------------------------------------- |
| `ping-streaming.pcapng`                   | `veth-pc1`, `wlan0` | `icmp \|\| dns` | Трансляцію адреси PC2 при `ping google.com`           |
| `local-connection-troubleshooting.pcapng` | `veth-pc1`          | `arp \|\| icmp` | ARP і ping між PC1 та PC2                             |
| `pc2-dns-troubleshooting.pcapng`          | `veth-pc1`          | `icmp`          | Зв'язок PC2 з DNS-серверами 192.168.50.1 та 8.8.8.8   |
| `defect-localization.pcapng`              | `veth-pc1`, `wlan0` | `icmp \|\| arp` | Дефект BUG-NAT-001: запити PC2 не виходять на `wlan0` |

## Результати

### ping-streaming.pcapng - трансляція адреси

Корисні фільтри: `icmp.ident == 38484` (той самий запит на обох інтерфейсах), `dns.qry.name == "google.com"`.

| seq | `veth-pc1` (до NAT)                   | `wlan0` (після NAT)                      | RTT     |
| --- | ------------------------------------- | ---------------------------------------- | ------- |
| 1   | 10.0.15.10 => 142.250.130.101, TTL 64 | 192.168.50.32 => 142.250.130.101, TTL 63 | 16,9 мс |
| 2   | 10.0.15.10 => 142.250.130.101, TTL 64 | 192.168.50.32 => 142.250.130.101, TTL 63 | 32,9 мс |
| 3   | 10.0.15.10 => 142.250.130.101, TTL 64 | 192.168.50.32 => 142.250.130.101, TTL 63 | 29,8 мс |
| 4   | 10.0.15.10 => 142.250.130.101, TTL 64 | 192.168.50.32 => 142.250.130.101, TTL 63 | 31,9 мс |
| 5   | 10.0.15.10 => 142.250.130.101, TTL 64 | 192.168.50.32 => 142.250.130.101, TTL 63 | 21,3 мс |

Спостереження:

- Ті самі ICMP-запити (id = 38484, seq 1-5) на `veth-pc1` мають джерело `10.0.15.10`, а на `wlan0` - `192.168.50.32`: MASQUERADE підміняє адресу джерела.
- Відповіді на `wlan0` адресовані `192.168.50.32`, а на `veth-pc1` - вже `10.0.15.10`: зворотна заміна за записом conntrack.
- TTL запиту зменшується з 64 до 63, а TTL відповіді - з 114 до 113: PC1 маршрутизує пакети (`ip_forward = 1`).
- Ідентифікатор ICMP зберігається (38484), оскільки на зовнішній адресі немає іншого сеансу з таким id.
- Різниця часу між кадрами на `veth-pc1` і `wlan0` - менше 0,1 мс, тобто трансляція не додає помітної затримки.
- DNS-запит PC2 `google.com` (10.0.15.10 => 192.168.50.1) також проходить через NAT і на `wlan0` має джерело `192.168.50.32`.

### local-connection-troubleshooting.pcapng - локальний зв'язок PC1 <=> PC2

| Напрямок              | ARP                                                               | ICMP                  | RTT          |
| --------------------- | ----------------------------------------------------------------- | --------------------- | ------------ |
| PC1 => PC2 (id 36610) | `Who has 10.0.15.10? Tell 10.0.15.1` => `is at fa:06:cd:5f:6c:99` | 4/4 відповіді, TTL 64 | 0,01-0,04 мс |
| PC2 => PC1 (id 40815) | `Who has 10.0.15.1? Tell 10.0.15.10` => `is at f2:da:32:a2:72:fe` | 4/4 відповіді, TTL 64 | 0,03-0,05 мс |

- Перед першим ping кожна сторона розв'язує MAC-адресу іншої через ARP - канальний рівень працює.

### pc2-dns-troubleshooting.pcapng - зв'язок PC2 з DNS

| Призначення  | ICMP id | Відповіді | TTL відповіді | RTT          |
| ------------ | ------- | --------- | ------------- | ------------ |
| 192.168.50.1 | 44360   | 4/4       | 63            | 4,4-25,9 мс  |
| 8.8.8.8      | 44385   | 4/4       | 116           | 16,1-37,7 мс |

- У стовпцях Source/Destination на `veth-pc1` видно лише внутрішню адресу `10.0.15.10`: трансляція відбувається далі, на `wlan0`.
- TTL відповіді роутера 63 (а не 64) - пакет пройшов через PC1 як через маршрутизатор.

### defect-localization.pcapng - локалізація дефекту

| Час (с)   | Інтерфейс  | Подія                                                         |
| --------- | ---------- | ------------------------------------------------------------- |
| 16,3-19,4 | `veth-pc1` | ping 10.0.15.10 => 10.0.15.1 (id 45488): 4/4 відповіді        |
| 22,9-26,0 | `veth-pc1` | ping 10.0.15.10 => 8.8.8.8 (id 45494): 4 запити, 0 відповідей |
| 22,9-26,0 | `wlan0`    | жодного ICMP-пакета до 8.8.8.8 (з 580 кадрів на `wlan0`)      |

- Перевірка: фільтр `ip.addr == 8.8.8.8 && icmp` показує 4 кадри, усі на `veth-pc1`;
  `frame.interface_name == "wlan0" && icmp` - порожньо.
- Локальний зв'язок з PC1 є, а пакети в Інтернет PC1 відкидає ще до NAT: маршрутизацію вимкнено (`ip_forward = 0`).

## Defect-report

**BUG-NAT-001.** Після перезавантаження PC1 клієнт внутрішньої мережі втрачає доступ до Інтернету: параметр маршрутизації з `/etc/sysctl.conf` не застосовується в Arch Linux.

- **Severity / Priority:** Major / High
- **Очікувано:** після перезавантаження `net.ipv4.ip_forward = 1`, ping 8.8.8.8 з PC2 успішний.
- **Фактично:** `ip_forward = 0`; ping 10.0.15.1 з PC2 - 4/4, ping 8.8.8.8 - 100% packet loss; запити є на `veth-pc1`, але відсутні на `wlan0`.
- **Причина:** `systemd-sysctl` читає лише `/etc/sysctl.d/`, `/run/sysctl.d/`, `/usr/lib/sysctl.d/`; інструкція до ЛР орієнтована на Debian/Ubuntu.
- **Рекомендації:** зберігати параметр у `/etc/sysctl.d/30-ipforward.conf`, правила iptables - у `/etc/iptables/iptables.rules` з увімкненою `iptables.service`.
