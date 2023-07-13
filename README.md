# VPN
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/vpn-hw22.git`
В текущей директории появится папка с именем репозитория. В данном случае vpn-hw22. Ознакомимся с содержимым:
```
cd vpn-hw22
ls -l
README.md
ansible
Vagrantfile
```
Здесь:
- ansible - папка с плэйбуками
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```

## TUN/TAP режимы VPN
Трубется установить пакет, настроить и запустить openvpn-server:
```
---
  # Отключаем SElinux
  - name: Put SELinux in permissive mode, logging actions that would be blocked.
    ansible.posix.selinux:
      policy: targeted
      state: permissive

  # Запускам и добавляем firewall
  - name: start firewalld
    service:
      name: firewalld
      state: started
      enabled: true

 # Добавляем интерфейсы в зону
  - name: add eth0 to public
    ansible.posix.firewalld:
      zone: public
      interface: eth0
      permanent: true
      state: enabled
    tags:
      - firewall_conf

 # Добавляем интерфейсы в зону
  - name: add eth1 to public
    ansible.posix.firewalld:
      zone: public
      interface: eth1
      permanent: true
      state: enabled
    tags:
      - firewall_conf

 # Добавляем интерфейсы в зону
  - name: add tap0 to public
    ansible.posix.firewalld:
      zone: public
      interface: tap0
      permanent: true
      state: enabled
    tags:
      - firewall_conf

  # Добавляем порт
  - ansible.posix.firewalld:
      port: 1209/udp
      permanent: true
      state: enabled

  # Устанавливаем EPEL
  - name: Install EPEL
    yum:
      name:
      - epel-release
      state: present
      update_cache: true
  
  # Обновляем пакеты и устанавливаем пакеты
  - name: install packages
    yum:
      name: 
        - openvpn
        - iperf3
      state: present

  # Создаём файл-ключ
  - name: Create Static KEY
    ansible.builtin.shell: openvpn --genkey --secret /etc/openvpn/static.key

  # Копируем статический ключ
  - name: Fetch the static.key
    run_once: yes
    fetch: src=/etc/openvpn/static.key dest=/tmp/ flat=yes

  # Включаем маршрутизацию транзитных пакетов
  - name: set up forward packages
    sysctl:
      name: net.ipv4.conf.all.forwarding
      value: '1'
      state: present

  # Копируем файл server1.conf на openvpn-server, указываем владельца и права
  - name: base set up TAP OpenVPN server 
    template:
      src: server1.conf
      dest: /etc/openvpn/server1.conf
      owner: root
      group: root
      mode: 0640

  # Создадим service unit для запуска openvpn
  - name: create service unit 
    template:
      src: openvpn.service
      dest: /etc/systemd/system/openvpn@.service
      owner: root
      group: root
      mode: 0640

  # Перезагрузка правил файрволла
  - name: Reload firewall
    ansible.builtin.shell: firewall-cmd --reload

  # Перезапускам и добавляем в автозагрузку
  - name: restart openvpn@server1
    service:
      name: openvpn@server1
      state: restarted
      enabled: true
      daemon_reload: yes

  # Перезагрузка правил файрволла
  - name: Reload firewall
    ansible.builtin.shell: firewall-cmd --reload
...
$ cat /etc/openvpn/server1.conf
port 1209
proto udp
dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
providers legacy default
allow-compression yes
comp-lzo
status /var/log/openvpn-status_server1.log
log /var/log/openvpn1.log
verb 3
```
Далее настраиваем openvpn-client:
```
---
  # Отключаем SElinux
  - name: Put SELinux in permissive mode, logging actions that would be blocked.
    ansible.posix.selinux:
      policy: targeted
      state: permissive

  # Устанавливаем EPEL
  - name: Install EPEL
    yum:
      name:
      - epel-release
      state: present
      update_cache: true
  
  # Обновляем пакеты и устанавливаем пакеты
  - name: install packages
    yum:
      name: 
        - openvpn
        - iperf3
      state: present

  # Копируем файл-ключ на клиент
  - name: Copy the Static KEY from server to client
    copy: 
      src: /tmp/static.key
      dest: /etc/openvpn/static.key

  # Копируем файл client1.conf на openvpn-client, указываем владельца и права
  - name: base set up TAP OpenVPN client 
    template:
      src: client1.conf
      dest: /etc/openvpn/client1.conf
      owner: root
      group: root
      mode: 0640

  # Создадим service unit для запуска openvpn
  - name: create service unit 
    template:
      src: openvpn.service
      dest: /etc/systemd/system/openvpn@.service
      owner: root
      group: root
      mode: 0640

  # Перезапускам и добавляем в автозагрузку
  - name: restart openvpn@client1
    service:
      name: openvpn@client1
      state: restarted
      enabled: true
      daemon_reload: yes

  # Перезапускам и добавляем в автозагрузку
  - name: restart openvpn@server1
    service:
      name: openvpn@server1
      state: restarted
      enabled: true
      daemon_reload: yes
    delegate_to: server

  # Проверка доступности
  - name: Test reachability to 10.10.10.1 using openvpn-client
    shell: ping -c 4 10.10.10.1

  # Останавливаем для проверки других vpn-туннелей
  - name: stop openvpn@client1
    service:
      name: openvpn@client1
      state: stopped
...
```
Разница в загрузке и передаче данных:
```
TAP:
$ iperf3 -c 10.10.10.1 -t 40 -i 5
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   630 MBytes   132 Mbits/sec  647             sender
[  5]   0.00-40.04  sec   630 MBytes   132 Mbits/sec                  receiver

iperf Done.
------------------------------------------------------------------------------
TUN:
$ iperf3 -c 10.20.10.1 -t 40 -i 5
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   640 MBytes   134 Mbits/sec  611             sender
[  5]   0.00-40.05  sec   639 MBytes   134 Mbits/sec                  receiver

iperf Done.
```
## RAS на базе OpenVPN
Трубется установить пакет, создать сертификаты, настроить и запустить openvpn-server:
```
---
  # Отключаем SElinux
  - name: Put SELinux in permissive mode, logging actions that would be blocked.
    ansible.posix.selinux:
      policy: targeted
      state: permissive

  # Запускам и добавляем firewall
  - name: start firewalld
    service:
      name: firewalld
      state: started
      enabled: true

 # Добавляем интерфейсы в зону
  - name: add eth0 to public
    ansible.posix.firewalld:
      zone: public
      interface: eth0
      permanent: true
      state: enabled

 # Добавляем интерфейсы в зону
  - name: add tun1 to public
    ansible.posix.firewalld:
      zone: public
      interface: tun1
      permanent: true
      state: enabled
    tags:
      - firewall_conf

 # Добавляем интерфейсы в зону
  - name: add eth1 to public
    ansible.posix.firewalld:
      zone: public
      interface: eth1
      permanent: true
      state: enabled

  # Добавляем порт
  - ansible.posix.firewalld:
      port: 1207/udp
      permanent: true
      state: enabled

  # Устанавливаем EPEL
  - name: Install EPEL
    yum:
      name:
      - epel-release
      state: present
      update_cache: true
  
  # Обновляем пакеты и устанавливаем пакеты
  - name: install packages
    yum:
      name: 
        - openvpn
        - iperf3
        - easy-rsa
        - expect
      state: present

  # Создаем RAS сертификаты
  - name: Create RAS KEYS
    ansible.builtin.shell: |
      set timeout 300
      cd /etc/openvpn/
      spawn rm -rf /etc/openvpn/pki
      spawn /usr/share/easy-rsa/3.0.8/easyrsa init-pki
      expect "/root/test/pki\n"
      spawn /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass
      expect "Enter PEM pass phrase:"
      send "rasvpn\n"
      expect "Verifying - Enter PEM pass phrase:"
      send "rasvpn\n"
      expect "\[Easy-RSA CA\]:"
      send "\n"
      expect "/pki\n"
      spawn /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass
      expect ":"
      send "\n"
      expect "server.key\n"
      spawn /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server
      expect "Confirm request details: "
      send "yes\n"
      expect ":"
      send "rasvpn\n"
      expect "server.crt\n"
      set timeout 3000
      spawn /usr/share/easy-rsa/3/easyrsa gen-req client nopass
      expect ":"
      send "\n"
      expect "client.key\n"
      spawn /usr/share/easy-rsa/3/easyrsa sign-req client client
      expect "Confirm request details: "
      send "yes\n"
      expect ":"
      send "rasvpn\n"
      expect "client.crt\n"
      spawn openvpn --genkey secret ca.key
      spawn /usr/share/easy-rsa/3.0.8/easyrsa gen-dh
      expect "dh.pem\n"
      exit 0
    args:
      executable: /usr/bin/expect

  # Копируем сертификаты
  - name: Fetch the ca.crt
    run_once: yes
    fetch: src=/etc/openvpn/pki/ca.crt dest=/tmp/ flat=yes
    tags:
      - fetch_certs

  # Копируем сертификаты
  - name: Fetch the client.crt
    run_once: yes
    fetch: src=/etc/openvpn/pki/issued/client.crt dest=/tmp/ flat=yes
    tags:
      - fetch_certs

  # Копируем сертификаты
  - name: Fetch the client.key
    run_once: yes
    fetch: src=/etc/openvpn/pki/private/client.key dest=/tmp/ flat=yes
    tags:
      - fetch_certs

  # Включаем маршрутизацию транзитных пакетов
  - name: set up forward packages
    sysctl:
      name: net.ipv4.conf.all.forwarding
      value: '1'
      state: present

  # Копируем файл server1.conf на openvpn-server, указываем владельца и права
  - name: base set up TAP OpenVPN server 
    template:
      src: server3.conf
      dest: /etc/openvpn/server3.conf
      owner: root
      group: root
      mode: 0640

  # Добавление роута
  - name: Route adding
    ansible.builtin.shell: echo 'iroute 10.10.10.0 255.255.255.0' > /etc/openvpn/client/client

  # Создадим service unit для запуска openvpn
  - name: create service unit 
    template:
      src: openvpn.service
      dest: /etc/systemd/system/openvpn@.service
      owner: root
      group: root
      mode: 0640

  # Перезагрузка правил файрволла
  - name: Reload firewall
    ansible.builtin.shell: firewall-cmd --reload

  # Перезапускам и добавляем в автозагрузку
  - name: restart openvpn@server3
    service:
      name: openvpn@server3
      state: restarted
      enabled: true
      daemon_reload: yes

  # Перезагрузка правил файрволла
  - name: Reload firewall
    ansible.builtin.shell: firewall-cmd --reload
...
```
Далее настраиваем openvpn-client:
```
---
  # Отключаем SElinux
  - name: Put SELinux in permissive mode, logging actions that would be blocked.
    ansible.posix.selinux:
      policy: targeted
      state: permissive

  # Устанавливаем EPEL
  - name: Install EPEL
    yum:
      name:
      - epel-release
      state: present
  #    update_cache: true
  
  # Обновляем пакеты и устанавливаем пакеты
  - name: install packages
    yum:
      name: 
        - openvpn
        - iperf3
      state: present

  # Копируем сертифкаты на клиент
  - name: Copy the ca.crt from server to client
    copy: 
      src: /tmp/ca.crt
      dest: /etc/openvpn/
      owner: root
      group: root
      mode: 0600

  # Копируем сертифкаты на клиент
  - name: Copy the client.crt from server to client
    copy: 
      src: /tmp/client.crt
      dest: /etc/openvpn/
      owner: root
      group: root
      mode: 0600

  # Копируем сертифкаты на клиент
  - name: Copy the client.key from server to client
    copy: 
      src: /tmp/client.key
      dest: /etc/openvpn/
      owner: root
      group: root
      mode: 0600

  # Копируем файл client1.conf на openvpn-client, указываем владельца и права
  - name: base set up TAP OpenVPN client 
    template:
      src: client3.conf
      dest: /etc/openvpn/client3.conf
      owner: root
      group: root
      mode: 0640

  # Создадим service unit для запуска openvpn
  - name: create service unit 
    template:
      src: openvpn.service
      dest: /etc/systemd/system/openvpn@.service
      owner: root
      group: root
      mode: 0640

  # Перезапускам и добавляем в автозагрузку
  - name: restart openvpn@server3
    service:
      name: openvpn@server3
      state: restarted
      enabled: true
      daemon_reload: yes
    delegate_to: server

  # Перезапускам и добавляем в автозагрузку
  - name: restart openvpn@client3
    service:
      name: openvpn@client3
      state: restarted
      enabled: true
      daemon_reload: yes

  # Проверка доступности
  - name: Test reachability to 10.20.30.1 using openvpn-client
    shell: ping -c 4 10.20.30.1

  # Останавливаем для проверки других vpn-туннелей
  - name: stop openvpn@client3
    service:
      name: openvpn@client3
      state: stopped
...
```
Проверка канала:
```
$ iperf3 -c 10.20.30.1 -t 40 -i 5
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   955 MBytes   200 Mbits/sec  586             sender
[  5]   0.00-40.06  sec   954 MBytes   200 Mbits/sec                  receiver

iperf Done.
```

## Реализовать OpenConnect сервер
Трубется установить пакет, создать сертификаты, настроить и запустить ocserv:
```
---
  # Отключаем SElinux
  - name: Put SELinux in permissive mode, logging actions that would be blocked.
    ansible.posix.selinux:
      policy: targeted
      state: permissive

  # Копируем файл server1.conf на openvpn-server, указываем владельца и права
  - name: base set up rules for firewalld
    template:
      src: direct.xml
      dest: /etc/firewalld/direct.xml
      owner: root
      group: root
      mode: 0640
    tags:
      - firewall_conf

  # Запускам и добавляем firewall
  - name: start firewalld
    service:
      name: firewalld
      state: started
      enabled: true
    tags:
      - firewall_conf

 # Добавляем интерфейсы в зону
  - name: add eth0 to public
    ansible.posix.firewalld:
      zone: public
      interface: eth0
      permanent: true
      state: enabled
    tags:
      - firewall_conf

 # Добавляем интерфейсы в зону
  - name: add eth1 to public
    ansible.posix.firewalld:
      zone: public
      interface: eth1
      permanent: true
      state: enabled
    tags:
      - firewall_conf

 # Добавляем интерфейсы в зону
  - name: add vpns1 to public
    ansible.posix.firewalld:
      zone: public
      interface: vpns1
      permanent: true
      state: enabled
    tags:
      - firewall_conf

  - name: enable masquerade
    ansible.posix.firewalld:
      masquerade: true
      state: enabled
      permanent: true
      zone: public
    tags:
      - firewall_conf

  # Добавляем порт
  - ansible.posix.firewalld:
      port: 443/tcp
      permanent: true
      state: enabled

  # Устанавливаем EPEL
  - name: Install EPEL
    yum:
      name:
      - epel-release
      state: present
      update_cache: true
  
  # Обновляем пакеты и устанавливаем пакеты
  - name: install packages
    yum:
      name: 
        - ocserv
        - python3-cryptography
      state: present
    tags:
      - cert_create

  # Включаем маршрутизацию транзитных пакетов
  - name: set up forward packages
    sysctl:
      name: net.ipv4.conf.all.forwarding
      value: '1'
      state: present

  - name: Ensure private key is present
    community.crypto.openssl_privatekey:
      path: /etc/pki/ocserv/private/server-key.pem
      size: 2048
      mode: 0600
      type: RSA
    tags:
      - cert_create

  - name: Ensure self-signed cert is present
    community.crypto.x509_certificate:
      path: /etc/pki/ocserv/public/server-cert.pem
      privatekey_path: /etc/pki/ocserv/private/server-key.pem
      provider: selfsigned
      selfsigned_not_after: "+3650d" # this is the default
      mode: 0644
    tags:
      - cert_create

  # Копируем файл server1.conf на openvpn-server, указываем владельца и права
  - name: base set up OCServ
    template:
      src: ocserv.conf
      dest: /etc/ocserv/ocserv.conf
      owner: root
      group: root
      mode: 0640

  - name: Create openconnect user
    ansible.builtin.shell: echo "vagrant" | ocpasswd altyn -c /etc/ocserv/ocpasswd

  # Перезагрузка правил файрволла
  - name: Reload firewall
    ansible.builtin.shell: firewall-cmd --reload

  # Перезапускам и добавляем в автозагрузку
  - name: restart ocserv
    service:
      name: ocserv
      state: restarted
      enabled: true
      daemon_reload: yes

  # Перезагрузка правил файрволла
  - name: Reload firewall
    ansible.builtin.shell: firewall-cmd --reload
...
```
Далее необходимо реализовать подключение с помощью openconnect-client:
```
---
  # Отключаем SElinux
  - name: Put SELinux in permissive mode, logging actions that would be blocked.
    ansible.posix.selinux:
      policy: targeted
      state: permissive

  # Устанавливаем EPEL
  - name: Install EPEL
    yum:
      name:
      - epel-release
      state: present
      update_cache: true
  
  # Обновляем пакеты и устанавливаем пакеты
  - name: install packages
    yum:
      name: 
        - openconnect
      state: present

  - name: openconnect connection
    ansible.builtin.shell: echo -e "yes\nvagrant\n" | openconnect -b -u altyn 192.168.50.10 >/dev/null 2>&1 &
    async: 10
    poll: 0    

  # Проверка доступности
  - name: Test reachability to 8.8.8.8 using openconnect
    shell: ping -c 4 8.8.8.8

  # Останавливаем для проверки других vpn-туннелей
  - name: close openconnect connection
    ansible.builtin.shell: killall -SIGINT openconnect
...
```
Проверяем:
```
$ iperf3 -c 10.12.0.1 -t 40 -i 5
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   677 MBytes   142 Mbits/sec  1216             sender
[  5]   0.00-40.10  sec   675 MBytes   141 Mbits/sec                  receiver

iperf Done.
```