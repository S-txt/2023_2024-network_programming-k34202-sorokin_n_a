# Лабораторная работа №3 Развертывание Netbox, сеть связи как источник правды в системе технического учета Netbox
---
University: [ITMO University](https://itmo.ru/ru/)
Faculty: [FICT](https://fict.itmo.ru)
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)
Year: 2023/2024
Group: K34202
Author: Sorokin Nikita Alekseevich
Lab: Lab3
Date of create: 28.11.2023
Date of finished: 3.12.2023
# Цель работы
С помощью Ansible и Netbox собрать всю возможную информацию об устройствах и сохранить их в отдельном файле.
# Ход работы
## Установка и конфигурация Netbox
0. В VirtualBox была создана дополнительная виртуальная машина для конфигурации Netbox
1. Установим PostgresSQL
```
sudo apt-get install -y postgresql libpq-dev
```
2. Подключимся к PostgreSQL
```
sudo -u postgres psql
```
3. Создадим базу данных и суперпользователя
```
CREATE DATABASE netbox;
CREATE USER admin WITH PASSWORD 'admin';
GRANT ALL PRIVILEGES ON DATABASE netbox TO admin;
```
*Важно, чтобы команды SQL оканчивались `;`*
4. Установим Redis
```
sudo apt-get install -y redis-server
```
5. Установим библиотеки python
```
sudo apt-get install -y python3 python3-pip python3-venv python3-dev build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev
```
6. Скачаем архив netbox
```
wget https://github.com/netbox-community/netbox/archive/v3.4.1.tar.gz
tar -xzf v3.4.1.tar.gz -C /opt
```
7. Добавим пользователя
```
groupadd --system netbox
adduser --system --gid 996 netbox
chown --recursive netbox /opt/netbox/netbox/media/
```
8. Создадим виртуальное окружение python 
```
python3 -m venv /opt/netbox/venv
source venv/bin/activate
pip3 install -r requirements.txt
```
9. Сгенерируем секретный ключ
```
python3 netbox/generate_secret_key.py
```
10. Отредактируем конфигурационный файл, укажем `ALLOWED_HOSTS`, `DATABASE` и `SECRET_KEY`:
```
cp configuration.example.py configuration.py
```
11. Применим миграции базы данных
```
source venv/bin/activate
python3 manage.py migrate
```
12. Создадим суперпользователя
```
python3 manage.py createsuperuser
```
13. Соберем статистику:
```
python3 manage.py collectstatic --no-input
```
14. Установим Ansible модули для Netbox
```
ansible-galaxy collection install netbox.netbox
```
15. Для доступа к веб интерфейсу установим Ngix
```
sudo apt-get install -y nginx
sudo cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
cd /etc/nginx/sites-enabled/
sudo rm default
sudo ln -s /etc/nginx/sites-available/netbox
sudo nginx -t
sudo nginx -s reload
sudo cp contrib/gunicorn.py /opt/netbox/gunicorn.py
sudo cp contrib/*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl start netbox netbox-rq
sudo systemctl enable netbox netbox-rq
```
16. Запустим сервер
```
python3 manage.py runserver 0.0.0.0:8000 --insecure
```
![[src/Pasted image 20231203043404.png]] 
17. В веб интерфейсе создадим сайт, мануфактуру и роль. Создадим 2 роутера и интерфейс для них.
![[Pasted image 20231203043721.png]]
## Сбор данных Netbox
1. Создадим файл `netbox_inventory.yml`
```YAML
plugin: netbox.netbox.nb_inventory
api_endpoint: http://127.0.0.1:8000
token: <netbox account token>
validate_certs: True
config_context: False
interfaces: True
```
*Токен можно получить в веб интерфейсе Netbox*
2. Выполним скрипт и перенаправим результат в файл
```
ansible-inventory -v --list -y -i netbox_inventory.yml > nb_inventory_result.yml
```
*Полученный в результате YAML файл можно использовать как файл инвентаря при конфигурации Ansible*
```
all:
  vars:
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: community.routeros.routeros
    ansible_ssh_user: admin
    ansible_ssh_pass: admin
    ansible_ssh_port: 22
  children:
    ungrouped:
      hosts:
        Mikrotik_1:
          ansible_host: 10.10.10.5
          custom_fields: {}
          device_roles:
          - router
          device_types:
          - router
          interfaces:
          - _occupied: false
            bridge: null
            cable: null
            cable_end: ''
            connected_endpoints: null
            connected_endpoints_reachable: null
            connected_endpoints_type: null
            count_fhrp_groups: 0
            count_ipaddresses: 2
            created: '2023-11-28T06:03:29.222527Z'
            custom_fields: {}
            description: ''
            device:
              display: Mikrotik_1
              id: 1
              name: Mikrotik_1
              url: http://127.0.0.1:8000/api/dcim/devices/1/
            display: ether2
            duplex: null
            enabled: true
            id: 1
            ip_addresses:
            - address: 10.10.10.5/24
              comments: ''
              created: '2023-11-28T06:21:23.342109Z'
              custom_fields: {}
              description: ''
              display: 10.10.10.5/24
              dns_name: ''
              family:
                label: IPv4
                value: 4
              id: 4
              last_updated: '2023-11-28T06:21:23.342130Z'
              nat_inside: null
              nat_outside: []
              role: null
              status:
                label: Active
                value: active
              tags: []
              tenant: null
              url: http://127.0.0.1:8000/api/ipam/ip-addresses/4/
              vrf: null
            - address: 10.10.10.15/24
              comments: ''
              created: '2023-11-28T06:05:43.068083Z'
              custom_fields: {}
              description: ''
              display: 10.10.10.15/24
              dns_name: ''
              family:
                label: IPv4
                value: 4
              id: 1
              last_updated: '2023-11-28T06:05:43.068104Z'
              nat_inside: null
              nat_outside: []
              role:
                label: Secondary
                value: secondary
              status:
                label: Active
                value: active
              tags: []
              tenant: null
              url: http://127.0.0.1:8000/api/ipam/ip-addresses/1/
              vrf: null
            l2vpn_termination: null
            label: ''
            lag: null
            last_updated: '2023-11-28T06:03:29.222552Z'
            link_peers: []
            link_peers_type: null
            mac_address: null
            mark_connected: false
            mgmt_only: false
            mode: null
            module: null
            mtu: null
            name: ether2
            parent: null
            poe_mode: null
            poe_type: null
            rf_channel: null
            rf_channel_frequency: null
            rf_channel_width: null
            rf_role: null
            speed: null
            tagged_vlans: []
            tags: []
            tx_power: null
            type:
              label: 100BASE-TX (10/100ME)
              value: 100base-tx
            untagged_vlan: null
            url: http://127.0.0.1:8000/api/dcim/interfaces/1/
            vdcs: []
            vrf: null
            wireless_lans: []
            wireless_link: null
            wwn: null
          is_virtual: false
          local_context_data:
          - null
          locations: []
          manufacturers:
          - mymanufacturer
          primary_ip4: 10.10.10.5
          regions: []
          services: []
          site_groups: []
          sites:
          - mysite
          status:
            label: Active
            value: active
          tags: []
        Mikrotik_2:
          ansible_host: 10.10.10.3
          custom_fields: {}
          device_roles:
          - router
          device_types:
          - router
          interfaces:
          - _occupied: false
            bridge: null
            cable: null
            cable_end: ''
            connected_endpoints: null
            connected_endpoints_reachable: null
            connected_endpoints_type: null
            count_fhrp_groups: 0
            count_ipaddresses: 2
            created: '2023-11-28T06:03:45.455291Z'
            custom_fields: {}
            description: ''
            device:
              display: Mikrotik_2
              id: 2
              name: Mikrotik_2
              url: http://127.0.0.1:8000/api/dcim/devices/2/
            display: ether2
            duplex: null
            enabled: true
            id: 2
            ip_addresses:
            - address: 10.10.10.3/24
              comments: ''
              created: '2023-11-28T06:20:53.558988Z'
              custom_fields: {}
              description: ''
              display: 10.10.10.3/24
              dns_name: ''
              family:
                label: IPv4
                value: 4
              id: 3
              last_updated: '2023-11-28T06:20:53.559012Z'
              nat_inside: null
              nat_outside: []
              role: null
              status:
                label: Active
                value: active
              tags: []
              tenant: null
              url: http://127.0.0.1:8000/api/ipam/ip-addresses/3/
              vrf: null
            - address: 10.10.10.13/24
              comments: ''
              created: '2023-11-28T06:06:37.673109Z'
              custom_fields: {}
              description: ''
              display: 10.10.10.13/24
              dns_name: ''
              family:
                label: IPv4
                value: 4
              id: 2
              last_updated: '2023-11-28T06:06:37.673134Z'
              nat_inside: null
              nat_outside: []
              role:
                label: Secondary
                value: secondary
              status:
                label: Active
                value: active
              tags: []
              tenant: null
              url: http://127.0.0.1:8000/api/ipam/ip-addresses/2/
              vrf: null
            l2vpn_termination: null
            label: ''
            lag: null
            last_updated: '2023-11-28T06:03:45.455312Z'
            link_peers: []
            link_peers_type: null
            mac_address: null
            mark_connected: false
            mgmt_only: false
            mode: null
            module: null
            mtu: null
            name: ether2
            parent: null
            poe_mode: null
            poe_type: null
            rf_channel: null
            rf_channel_frequency: null
            rf_channel_width: null
            rf_role: null
            speed: null
            tagged_vlans: []
            tags: []
            tx_power: null
            type:
              label: 100BASE-TX (10/100ME)
              value: 100base-tx
            untagged_vlan: null
            url: http://127.0.0.1:8000/api/dcim/interfaces/2/
            vdcs: []
            vrf: null
            wireless_lans: []
            wireless_link: null
            wwn: null
          is_virtual: false
          local_context_data:
          - null
          locations: []
          manufacturers:
          - mymanufacturer
          primary_ip4: 10.10.10.3
          regions: []
          services: []
          site_groups: []
          sites:
          - mysite
          status:
            label: Active
            value: active
          tags: []
```
3. Cоздалим Playbook для изменения имени пользователя и ip адреса
```
- name: Set CHR
  hosts: ungrouped
  tasks:
    - name: Set Name
      community.routeros.command:
        commands:
          - /system identity set name="{{interfaces[0].device.name}}"
    - name: Set IP
      community.routeros.command:
        commands:
        - /ip address add address="{{interfaces[0].ip_addresses[1].address}}" interface="{{interfaces[0].display}}"
```
4. Cоздалим Playbook для получения серийного номера CHR и установки его в профиль Netbox
```
- name: Serial Numbers
  hosts: ungrouped
  tasks:
    - name: Serial Number
      community.routeros.command:
        commands:
          - /system license print
      register: license_print
    - name: Name
      community.routeros.command:
        commands:
          - /system identity print
      register: identity_print
    - name: Serial Number to Netbox
      netbox_device:
        netbox_url: http://127.0.0.1:8000
        netbox_token: <API token>
        data:
          name: "{{identity_print.stdout_lines[0][0].split(' ').1}}"
          serial: "{{license_print.stdout_lines[0][0].split(' ').1}}"
```
![[src/Pasted image 20231203045245.png]]
![[src/Pasted image 20231203045316.png]]
# Вывод
В результате выполнения лабораторной работы была создана дополнительная виртуальная машина, на которой был развернут и сконфигурирован Netbox. Так же были написаны playbook для установки параметров на базе собранной информации Netbox и playbook получающий серийный номер chr и устанавливающий его в профиль Netbox.