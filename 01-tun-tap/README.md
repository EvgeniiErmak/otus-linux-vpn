# 1. VPN между двумя ВМ в режимах tun / tap

## Задание
Настроить VPN между двумя ВМ в tun/tap режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях.

## Схема сети

     +------------------------+                       +------------------------+
     |  server (server.loc)   |    192.168.56.0/24    |  client (client.loc)   |
     |  enp0s8 192.168.56.10  |<--------------------->|  enp0s8 192.168.56.20  |
     |  tap0/tun0 10.10.10.1  |== OpenVPN UDP/1194 ===|  tap0/tun0 10.10.10.2  |
     +------------------------+     (static key)      +------------------------+

## Реализация
- [Vagrantfile](Vagrantfile) — 2 ВМ `ubuntu/jammy64`, Ansible запускается после поднятия последней ВМ сразу на все хосты.
- [ansible/playbook.yml](ansible/playbook.yml) — установка `openvpn iperf3 selinux-utils`, `setenforce 0`, юнит `openvpn@.service`, генерация `static.key` на server и копирование на client.
- [ansible/templates](ansible/templates) — конфиги из методички, режим задаётся переменной `vpn_dev` (по умолчанию `tap`).

## Запуск

    vagrant up                                            # режим tap
    ansible-playbook ansible/playbook.yml -e vpn_dev=tun  # переключение в tun

Замер скорости:

    vagrant ssh server -c "iperf3 -s -D"
    vagrant ssh client -c "iperf3 -c 10.10.10.1 -t 40 -i 5"

## Результаты ([results](results))

| Режим | Bitrate sender / receiver | Retr | Ping avg |
|---|---|---|---|
| [tap](results/tap.txt) | 241 / 240 Mbit/s | 1952 | 1.11 ms |
| [tun](results/tun.txt) | 245 / 244 Mbit/s | 598 | 0.64 ms |

## Выводы
- Скорость почти одинаковая: tun быстрее на ~2% и даёт в 3 раза меньше ретрансмитов.
- Узкое место — однопоточное шифрование OpenVPN в userspace и виртуализация, а не режим туннеля.
- **tap (L2)** — передаёт Ethernet-кадры, поддерживает бриджинг, broadcast/multicast и не-IP протоколы. Минусы: лишний Ethernet-заголовок (14 байт) в каждом пакете, в туннель попадает широковещательный трафик (ARP и т.п.), не поддерживается на Android/iOS.
- **tun (L3)** — передаёт только IP-пакеты, меньше накладных расходов, проще маршрутизация, поддерживается всеми клиентами. Минус: нет L2 (бриджинга, broadcast, не-IP протоколов).
- Для удалённого доступа и связи сетей предпочтителен **tun**, tap нужен только когда требуется общий L2-сегмент.
