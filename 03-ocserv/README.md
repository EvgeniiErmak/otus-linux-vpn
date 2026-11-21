# 3. (*) OpenConnect-сервер (ocserv)

## Задание
Самостоятельно изучить и настроить ocserv, подключиться с хоста к ВМ.

## Схема сети

     +---------------------------+                         +---------------------------+
     |  Хост (otus-1)            |    192.168.56.0/24      |  ocserv (ocserv.loc)      |
     |  vboxnet4 192.168.56.1    |<----------------------->|  enp0s8 192.168.56.30     |
     |  tun0 10.20.20.190        |== TLS 443/tcp (CSTP) ===|  vpns0 10.20.20.1         |
     |  (openconnect)            |== DTLS 443/udp =========|  ipv4-network 10.20.20.0/24|
     +---------------------------+                         +---------------------------+

## Реализация
- [Vagrantfile](Vagrantfile) — 1 ВМ `ubuntu/jammy64`, настройка через Ansible.
- [ansible/playbook.yml](ansible/playbook.yml) — установка `ocserv gnutls-bin`, создание CA и сертификата сервера через `certtool`, пользователь `otus` в `/etc/ocserv/ocpasswd`, конфиг, запуск сервиса, копирование `ca-cert.pem` на хост в [client](client).
- [ansible/templates](ansible/templates) — шаблоны `certtool` ([ca.tmpl.j2](ansible/templates/ca.tmpl.j2), [server.tmpl.j2](ansible/templates/server.tmpl.j2)) и [ocserv.conf.j2](ansible/templates/ocserv.conf.j2).

Параметры:
- аутентификация `plain` (логин/пароль `otus` / `otus`);
- порт 443 TCP (CSTP) + 443 UDP (DTLS);
- сеть VPN 10.20.20.0/24;
- split tunnel: клиенту пушится только `route = 10.20.20.0/255.255.255.0`, весь остальной трафик хоста идёт мимо VPN.

## Запуск и проверка

    vagrant up
    apt install -y openconnect
    cd client
    echo 'otus' | openconnect --user=otus --passwd-on-stdin --cafile ca-cert.pem \
      -b --pid-file=/run/openconnect.pid 192.168.56.30
    ping -c 4 10.20.20.1
    vagrant ssh ocserv -c "sudo occtl show users"
    kill $(cat /run/openconnect.pid)      # отключение

Результаты: [openconnect.log](results/openconnect.log), [ip_addr.txt](results/ip_addr.txt), [ip_route.txt](results/ip_route.txt), [occtl.txt](results/occtl.txt), [ping.txt](results/ping.txt).

## Заметки
- В ocserv 1.1.3 (Ubuntu 22.04) с `isolate-workers = true` рабочий процесс падает при каждом подключении (`The futex facility returned an unexpected error code`): seccomp-песочница блокирует нужный системный вызов, и клиент видит обрыв TLS (`Error in the pull function`). Поэтому `isolate-workers = false`.
- `max-ban-score = 0` отключает бан IP за неудачные попытки входа, чтобы не заблокировать хост во время отладки.
- Без директивы `route` ocserv пушит маршрут по умолчанию (full tunnel), и весь трафик хоста, включая SSH, ушёл бы в VPN.
