# OTUS «Администратор Linux. Professional» — Домашнее задание: VPN

## Цель
Создать домашнюю сетевую лабораторию. Научиться настраивать VPN-сервер в Linux-based системах.

## Задание
1. Настроить VPN между двумя ВМ в tun/tap режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях.
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ.
3. (*) Самостоятельно изучить и настроить ocserv, подключиться с хоста к ВМ.

Формат сдачи: Vagrantfile + Ansible.

## Структура репозитория

| Каталог | Описание |
|---|---|
| [01-tun-tap](01-tun-tap) | OpenVPN со static key между двумя ВМ, режимы tap/tun, замеры iperf3 и выводы — [README](01-tun-tap/README.md) |
| [02-ras](02-ras) | RAS на OpenVPN с PKI easy-rsa, подключение с хоста по клиентскому сертификату — [README](02-ras/README.md) |
| [03-ocserv](03-ocserv) | (*) OpenConnect-сервер ocserv, подключение с хоста через openconnect — [README](03-ocserv/README.md) |

В каждом каталоге: `Vagrantfile`, `ansible.cfg`, `ansible/` (плейбук и шаблоны), `results/` (выводы команд проверки).

## Стенд
- Хост: Ubuntu 26.04, VirtualBox 7.2, Vagrant 2.4.9, Ansible core 2.20
- ВМ: `ubuntu/jammy64` (Ubuntu 22.04)
- Сеть ВМ: host-only 192.168.56.0/24

## Запуск
Задания запускаются по очереди (задания 1 и 2 используют один IP 192.168.56.10):

    cd 01-tun-tap && vagrant up      # затем vagrant destroy -f
    cd 02-ras     && vagrant up
    cd 03-ocserv  && vagrant up

## Итоги
- **tun vs tap:** скорость почти одинаковая (245 vs 241 Mbit/s), tun даёт меньше накладных расходов и ретрансмитов; подробности в [01-tun-tap/README.md](01-tun-tap/README.md#выводы).
- **RAS:** хост подключается по сертификату, сеть туннеля 10.10.10.0/24 импортируется в таблицу маршрутизации; отличия от методички описаны в [02-ras/README.md](02-ras/README.md#особенности-и-отличия-от-методички).
- **ocserv:** подключение по логину/паролю через TLS + DTLS, split tunnel; заметки в [03-ocserv/README.md](03-ocserv/README.md#заметки).
