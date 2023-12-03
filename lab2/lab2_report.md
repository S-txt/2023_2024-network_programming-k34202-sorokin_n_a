# Лабораторная работа №2 Развертывание дополнительного CHR, первый сценарий Ansible
---
University: [ITMO University](https://itmo.ru/ru/)
Faculty: [FICT](https://fict.itmo.ru)
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)
Year: 2023/2024
Group: K34202
Author: Sorokin Nikita Alekseevich
Lab: Lab2
Date of create: 27.11.2023
Date of finished: 3.12.2023

---
# Цель работы 
С помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory.

# Ход работы
## Создание второго CHR
1. С помощью функции клонирования была создана вторая виртуальная машина **CHR 2**
2. В веб-интерфейсе OpenVPN был добавлен дополнительный пользователь
3. Аналогично настройке OpenVPN в WinBox из прошлой лабораторной работы, на **CHR 2** был установлен сертификат и подключен интерфейс OpenVPN
4. В результате конфигурации оборудования мы получили сеть следующего вида
	![[src/Диаграмма без названия.drawio (1).png]]
## Настройка роутеров с помощью Ansible
1. Создадим файл конфигурации инвентаризации `hosts.ini`:
	``` ini
	[chr]
	chr1 ansible_host= <local LAN ip 1>
	chr2 ansible_host= <local LAN ip 2>
	
	[chr:vars]
	ansible_connection=ansible.netcommon.network_cli
	ansible_network_os=community.routeros.routeros
	ansible_user=admin
	ansible_ssh_pass=admin
```
2. Проверим правильность файла инвентаризации и доступность роутеров
	```
	ansible -i hosts.ini -m ping chr 
	```
3. Далее создадим Ansible-playbook, который будет проводить требуемую конфигурацию и сбор данных с роутеров
	```
	- name: Setup CHR
	  hosts: chr
	  tasks:
	    - name: Create Users
	      routeros_command:
	        commands: 
	          - /user add name=user group=read password=user
	
	    - name: NTP Client
	      routeros_command:
	        commands:
	          - /system ntp client set enabled=yes server=0.ru.pool.ntp.org
	        
	    - name: OSPF
	      routeros_command:
	        commands: 
	          - /interface bridge add name=loopback
	          - /ip address add address=<local subnet>.0.1 interface=loopback network=<local subnet>.0.1
	          - /routing id add disabled=no id=<local subnet>.0.1 name=OSPF_ID select-dynamic-id=""
	          - /routing ospf instance add name=ospf-1 originate-default=always router-id=OSPF_ID
	          - /routing ospf area add instance=ospf-1 name=backbone
	          - /routing ospf interface-template add area=backbone auth=md5 auth-key=admin interface=ether1
	
	    - name: Facts
	      routeros_facts:
	        gather_subset:
	          - interfaces
	      register: output_ospf
	
	    - name: Output
	      debug:
	        var: "output_ospf"
	```
4. Запустим созданный playbook:
	```
	ansible-playbook ansible-playbook.yml -i hosts.ini
	```
5. Проверим проведенные настройки Ansible
	![[src/Lab2_1.png]]
	![[src/Pasted image 20231203040958.png]]
# Вывод
В результате выполнения лабораторной работы была создана дополнительная виртуальная машина CHR 2, на ней проведены все настройки OpenVPN из лабораторной работы 1. Созданы два файла конфигураций Ansible: файл инвентаризации hosts.ini и Ansible playbook со списком команд для конфигурации роутеров и сбора данных