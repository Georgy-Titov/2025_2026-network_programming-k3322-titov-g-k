# Отчет по лабораторной работе №1 - "Развертывание дополнительного CHR, первый сценарий Ansible".

## Шапка отчета

* University: [ITMO University](https://itmo.ru/ru/)
* Faculty: [ФПиН](https://fpin.itmo.ru/ru)
* Course: [Network-programming](https://itmo-ict-faculty.github.io/network-programming/)
* Year: 2025/2026
* Group: K3322
* Author: Titov Georgy Konstantinovich
* Lab: Lab2
* Date of create: 25.05.2026
* Date of finished: 28.05.2026

## Описание

В данной лабораторной работе вы на практике ознакомитесь с системой управления конфигурацией Ansible, использующаяся для автоматизации настройки и развертывания программного обеспечения.

## Цель работы

С помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory.

## Задание
Ход работы:

Установить второй CHR на своем ПК.  
Организовать второй OVPN Client на втором CHR.  
Используя Ansible, настроить сразу на 2-х CHR:  

* логин/пароль;
* NTP Client;
* OSPF с указанием Router ID; 4. Собрать данные по OSPF топологии и полный конфиг устройства.

> Что использовать: routeros_facts, routeros_commands.

## Схема сети

<img width="611" height="586" alt="image" src="https://github.com/user-attachments/assets/4b969472-2a12-4776-adb9-4324055ffae7" />

## Выполнение работы

### Конфигурация второго роутера

В задании от нос требуется поднять второй роутер. Клонируем второй роутер, и уже в новом роуетере меняем необходимые конфиги:

```
ip address=10.220.220.3/22 interface=tunel # Присваиваем новый ip-adress

interface/wireguard/ remove 0 # Удаляем старый интерфейс wireguard
interface/wireguard/ add listen-port=8888 name=tunel # Создаем новый, чтобы получить новые ключи
```

В конфигурации на VPS добавляем нового пира:

```
[Interface]
Address = 10.220.220.1/24
ListenPort = 8888
PrivateKey = <VPS Wireguard privatekey>

[Peer]
# CHR-1
PublicKey = iRqpd7gKo67PJSnpQIJwoMDfljBi8N+XCECyNrSREVQ=
AllowedIPs = 10.220.220.2/32
PersistentKeepalive = 25

[Peer]
# CHR-2
PublicKey = m9UleLSEIJXuwhKedP9PZNRsS5jSStu1H4/bPDXTYFg=
AllowedIPs = 10.220.220.3/32
PersistentKeepalive = 25
```

Проверяем что все работает:

<img width="392" height="267" alt="image" src="https://github.com/user-attachments/assets/690cc122-6c4f-4139-8927-5c4ee423a7dd" />

## Ansible:

Готовая структруа проекта:

<img width="522" height="261" alt="image" src="https://github.com/user-attachments/assets/f32028e5-a198-4a99-ba26-5c108562f509" />

### Inventory

```

[routers]
CHR-1 ansible_host=10.220.220.2 loopback_ip=10.255.255.222 ospf_interface=ether1
CHR-2 ansible_host=10.220.220.3 loopback_ip=10.255.255.223 ospf_interface=ether2

[routers:vars]
ansible_user=admin
ansible_password=admin
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
```
Проверяем наш инвентори `ansible-inventory -i invetnory/hosts --list`:

<img width="1011" height="564" alt="image" src="https://github.com/user-attachments/assets/d0101f8b-e90c-4d01-bb76-16a7a692ffe7" />

Проверяем подключение Ansible к роутерам:

<img width="1025" height="573" alt="image" src="https://github.com/user-attachments/assets/0c9f084e-1610-43f6-b396-8692e526736f" />

### Playbooks

Далее создаем два плейбука `setup.yml` - для раскатки наших хостов, `export.yml` - для сбора информации с роутеров

> setup.yml:

```
- name: Configure CHR routers
  hosts: routers # На какие хоста катим
  gather_facts: false

  vars:
    ntp_server: 0.ru.pool.ntp.org #NTP server

  tasks:

    - name: Create automation user # Cоздамем пользака
      community.routeros.command:
        commands:
          - /user add name=georgy password=georgy group=full

    - name: Configure NTP client # Конфигурируем NTP server
      community.routeros.command:
        commands:
          - /system ntp client set enabled=yes servers={{ ntp_server }}

    - name: Configure loopback interface and OSPF # Конфигурируем OSPF
      community.routeros.command:
        commands:
          - /interface bridge add name=loopback
          - /ip address add address={{ loopback_ip }}/32 interface=loopback
          - /routing ospf instance add name=inst router-id={{ loopback_ip }}
          - /routing ospf area add name=backbone area-id=0.0.0.0 instance=inst
          - /routing ospf interface-template add area=backbone interfaces={{ ospf_interface }} type=ptp
```

> export.yml:

```

- name: Export CHR info
  hosts: routers
  gather_facts: false

  vars:
    export_dir: "{{ playbook_dir }}/../exports/lab2"

  tasks:

    - name: Get all CHR facts
      community.routeros.facts:
        gather_subset: all
      register: chr_facts

    - name: Get full configuration (/export)
      community.routeros.command:
        commands:
          - /export
      register: chr_export

    - name: Get OSPF neighbors
      community.routeros.command:
        commands:
          - /routing ospf neighbor print detail
      register: ospf_neighbors

    - name: Get OSPF interfaces
      community.routeros.command:
        commands:
          - /routing ospf interface print detail
      register: ospf_interfaces

    - name: Ensure export directory exists
      delegate_to: localhost
      run_once: true
      file:
        path: "{{ export_dir }}"
        state: directory

    - name: Save facts to file
      delegate_to: localhost
      copy:
        content: "{{ chr_facts | to_nice_json }}"
        dest: "{{ export_dir }}/{{ inventory_hostname }}_facts.json"

    - name: Save full config to file
      delegate_to: localhost
      copy:
        content: "{{ chr_export.stdout | join('\n') }}"
        dest: "{{ export_dir }}/{{ inventory_hostname }}_export.txt"

    - name: Save OSPF topology to file
      delegate_to: localhost
      copy:
        content: |
          OSPF NEIGHBORS:
          {{ ospf_neighbors.stdout | join('\n') }}

          OSPF INTERFACES:
          {{ ospf_interfaces.stdout | join('\n') }}
        dest: "{{ export_dir }}/{{ inventory_hostname }}_ospf.txt"

```

## Проверка работоспособности

При помощи следующей команды прокатываем ansible плейбуки - `ansible-playbook -i inventory/hosts playbooks/<playbook>.yml`:

<img width="1219" height="636" alt="image" src="https://github.com/user-attachments/assets/0a915845-0380-488a-abaf-99af598824bb" />

<img width="1215" height="708" alt="image" src="https://github.com/user-attachments/assets/6fe3b246-0ccc-4663-a9e5-c1e0d17789c1" />

Проверяем что юзеры прокатились:

<img width="1280" height="437" alt="image" src="https://github.com/user-attachments/assets/e5f66746-b447-4012-9776-7280269f2ea9" />

Проверка NTP-server:

<img width="1280" height="291" alt="image" src="https://github.com/user-attachments/assets/0dc65d8b-d175-4588-86de-9a523b211d42" />

Проверка OSPF:

<img width="1280" height="434" alt="image" src="https://github.com/user-attachments/assets/0a6e57d8-a36b-42c6-b9c3-41583730b17b" />

## Заключение:

В ходе выполнения лабораторной работы было выполнено развертывание второго MikroTik CHR и организовано подключение устройств через WireGuard VPN. После этого была настроена система автоматизации Ansible для централизованного управления сетевыми устройствами.

В процессе работы был создан inventory-файл, содержащий описание сетевых устройств, параметры подключения и индивидуальные переменные для каждого маршрутизатора. Также были разработаны playbook-сценарии для автоматической настройки MikroTik CHR и сбора информации с устройств.

С помощью Ansible на двух маршрутизаторах были автоматически выполнены следующие действия:

* создан пользователь с необходимыми правами доступа;
* настроен NTP-клиент для синхронизации времени;
* настроен OSPF с уникальными Router ID;
* создан loopback-интерфейс;
* выполнен сбор информации о соседях и интерфейсах OSPF;
* получен полный экспорт конфигурации устройств.

В результате выполнения лабораторной работы были получены файлы конфигураций маршрутизаторов, информация о топологии OSPF и результаты проверки сетевой связности между устройствами.
