# Лабораторная работа №1 Установка CHR и Ansible, настройка VPN
---
University: [ITMO University](https://itmo.ru/ru/)
Faculty: [FICT](https://fict.itmo.ru)
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)
Year: 2023/2024
Group: K34202
Author: Sorokin Nikita Alekseevich
Lab: Lab1
Date of create: 27.11.2023
Date of finished: 3.12.2023

---
# Цель работы 

Создание виртуальной машины на базе RouterOS. Создание на облачном хостинге Yandex Cloud виртуальной машины. Установка и настройка OpenVPN AC и Ansible

# Ход работы

## Создание виртуальной машины CHR
1. C официального сайта Mikrotik был скачан образ виртуального диска RouterOS CHR
2. В Oracle VirtualBox была создана виртуальная машина CHR на базе скачанного виртуального диска
![[src/Pasted image 20231203020802.png]]
3. Так же с официального сайта для удобства конфигурации скачиваем WinBox и подключаемся к запущенной виртуальной машине выбрав ее в списке **Neighbors**

## Создание виртуальной машины в YandexCloud
1. На локальной машине создаем ключ ssh для подключения к удаленной виртуальной машине:
	```
	ssh-keygen -t ed25519
	```
2. В личном кабинете Yandex Cloud создали прерываемую виртуальную машину на Ubuntu 22.04.
3. В пункте **Доступ** при создании виртуальной машины требуется указать сгенерированный публичный ключ SSH
4. После окончания процесcа создания виртуальной машины подключаемся к ней с помощью ssh
	```
	ssh -i ed25519 <username>@<vm public adress>
	```
	*После флага `-i` требуется указать место расположения сгенерированного ключа п.1*
	![[src/Pasted image 20231203021934.png]]
## Установка и настройка OpenVPN и Ansible
1. Обновим OS до актуального состояния
	```
	sudo apt update & sudo apt upgrade
	sudo do-release-upgrade
	```
2. Установим python и Ansible
	```
	sudo apt install python3-pip 
	sudo pip3 install ansible 
	```
3. Убедимся в установке Ansible
	```
	ansible --version
	```
4. Установим OpenVPN AC
	```
	wget https://as-repository.openvpn.net/as-repo-public.asc -qO /etc/apt/trusted.gpg.d/as-repository.asc
	echo "deb [arch=amd64 signed-by=/etc/apt/trusted.gpg.d/as-repository.asc] http://as-repository.openvpn.net/as/debian jammy main">/etc/apt/sources.list.d/openvpn-as-repo.list
	apt update && apt -y install openvpn-as
	```
	![[src/Pasted image 20231203030935.png]]
5. Переходим в веб интерфейс OpenVPN по адресу `httpsL//<vm public ip>:943/admin`
6. Входим в профиль администратора по полученным данным из п.4
7. Во вкладке **Network Settings** указываем **Protocol** -> **TCP** и порт `443`
8. Во вкладке **Advanced VPN** отключаем TLS
9. Во вкладке **User Permissions** создаем нового пользователя
10. Во вкладке **User Profile** добавляем конфигурацию созданному пользователю, скачиваем полученный файл.
## Подключение CHR к OpenVPN
1. Скачанный сертификат OpenVPN добавляем в виртуальную машину CHR, перетащив с помощью Drag&Drop в список файлов WinBox
2. Через терминал CHR импортируем сертификат
	```RouterOS
	certificate import file-name=<cert name>
	```
3. Далее был создан новый интерфейс-клиент OpenVPN со следующими настройками:
	![[src/Pasted image 20231203031946.png]]
4. Проверим подключение к удаленному серверу:
	![[src/Pasted image 20231203032051.png]]

# Вывод
В результате выполнения лабораторной работы было выполнено развертывание локальной виртуальной машины на базе Mikrotik RouterOS, удаленного сервера Ubuntu 22.04 на платформе Yandex Cloud. Скачаны и настроены OpenVPN AC и Ansible. Создан интерфейс подключения OpenVPN на роутере и проверена работоспособность VPN туннеля.