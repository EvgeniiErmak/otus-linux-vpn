# 2. RAS на базе OpenVPN с клиентскими сертификатами

## Задание
Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ.

## Схема сети

     +---------------------------+                         +---------------------------+
     |  Хост (otus-1)            |    192.168.56.0/24      |  server (server.loc)      |
     |  vboxnet4 192.168.56.1    |<----------------------->|  enp0s8 192.168.56.10     |
     |  tun0 10.10.10.6          |== OpenVPN UDP/1207 =====|  tun0 10.10.10.1          |
     |  (client.conf)            |    TLS, PKI easy-rsa    |  server 10.10.10.0/24     |
     +---------------------------+                         +---------------------------+

## Реализация
- [Vagrantfile](Vagrantfile) — 1 ВМ `ubuntu/jammy64`, настройка через Ansible.
- [ansible/playbook.yml](ansible/playbook.yml) — установка `openvpn easy-rsa`, `setenforce 0`, PKI (CA, сертификаты server/client, DH), `server.conf`, юнит `openvpn@.service`, копирование `ca.crt`, `client.crt`, `client.key` на хост в [client](client).
- [ansible/templates/server.conf.j2](ansible/templates/server.conf.j2) — конфиг сервера из методички.
- [client/client.conf](client/client.conf) — конфиг клиента на хосте.

## Запуск и проверка

    vagrant up
    cd client && openvpn --config client.conf   # окно 1
    ping -c 4 10.10.10.1                         # окно 2
    ip r

Результаты: [client-connect.log](results/client-connect.log), [ping.txt](results/ping.txt), [ip_route.txt](results/ip_route.txt).

## Особенности и отличия от методички
1. **Добавлен `easyrsa build-ca nopass`.** В методичке этого шага нет, но без CA не работает `sign-req` и не появляется `pki/ca.crt`, на который ссылается `server.conf`.
2. **Убран `iroute 10.10.10.0 255.255.255.0` для клиента.** iroute закрепляет всю сеть туннеля за клиентом, поэтому сервер вырезает маршрут из push (в логе `REMOVE PUSH ROUTE: 'route 10.10.10.0 255.255.255.0'`) и хост не получает маршрут к 10.10.10.1. Без iroute сервер пушит `route 10.10.10.0 255.255.255.0`, и сеть туннеля импортируется в таблицу маршрутизации хоста.
3. **У `route 192.168.56.0 255.255.255.0` указана метрика 1000.** Хост сам подключён к 192.168.56.0/24 через host-only адаптер. Без метрики маршрут через tun0 становился приоритетнее, и транспорт OpenVPN к 192.168.56.10 заворачивался в сам туннель (петля).
4. На хосте OpenVPN 2.7: `comp-lzo` и `persist-key` дают предупреждения deprecated, на работу это не влияет.
