# Отчет по лабораторной работе №3 - "Развертывание Netbox, сеть связи как источник правды в системе технического учета Netbox".

## Шапка отчета

* University: [ITMO University](https://itmo.ru/ru/)
* Faculty: [ФПиН](https://fpin.itmo.ru/ru)
* Course: [Network-programming](https://itmo-ict-faculty.github.io/network-programming/)
* Year: 2025/2026
* Group: K3322
* Author: Titov Georgy Konstantinovich
* Lab: Lab3
* Date of create: 03.06.2026
* Date of finished: 06.06.2026

## Описание

В данной лабораторной работе вы ознакомитесь с интеграцией Ansible и Netbox и изучите методы сбора информации с помощью данной интеграции.

## Цель работы

С помощью Ansible и Netbox собрать всю возможную информацию об устройствах и сохранить их в отдельном файле.

## Задание
Ход работы:

1. Поднять Netbox на дополнительной VM.
2. Заполнить всю возможную информацию о ваших CHR в Netbox.
3. Используя Ansible и роли для Netbox в тестовом режиме сохранить все данные из Netbox в отдельный файл, результат приложить в отчёт.
4. Написать сценарий, при котором на основе данных из Netbox можно настроить 2 CHR, изменить имя устройства, добавить IP адрес на устройство.
5. Написать сценарий, позволяющий собрать серийный номер устройства и вносящий серийный номер в Netbox.

## Выполнение работы

### Устанвока Netbox

> NetBox был поднят на новой виртуальной машине. Все сценарии были написаны на виртуальной машине, поднятой в предыдущей лабораторной работе. Также все устройства были связаны во внутреннюю сеть VirualBox, по следующей причние. Так как хост с Wireguard был поднят локально и он получал Ip-адесс через Bridge-addapter, то при каждом рестарте ему назначался новый локальный Ip-адрес из сети 192.168.0.0/24 - такое поведение ломало конфиги подключения к Wireguard peer на роутерах. Всем устройствам был назначен новый сетевой интерфейс с соответствующим Ip-адресом из сети 10.0.0.0/24.

<img width="960" height="280" alt="image" src="https://github.com/user-attachments/assets/f3830251-d305-4238-8717-62bfe340eab9" />

```
sudo apt update
sudo apt install -y git python3-venv python3-pip redis-server postgresql
python3 -m venv netbox-venv
source netbox-venv/bin/activate
sudo git clone https://github.com/netbox-community/netbox.git
cd netbox
pip install -r requirements.txt
cp netbox/netbox/configuration_example.py netbox/netbox/configuration.py
pip install ansible pynetbox netbox-api
```

Далее создаем базу данных postgresql:

```
sudo -u postgres psql

CREATE DATABASE netbox;
CREATE USER netbox WITH PASSWORD 'netboxpass';
GRANT ALL PRIVILEGES ON DATABASE netbox TO netbox;
ALTER DATABASE netbox OWNER TO netbox;
\q
```
В configuration.py указываем ALLOWED_HOSTS = ['*'].

Далее запускаем redis-server:

```
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

Также нам потреубется чуть поправить configuration.py:

```
В configuration.py указываем ALLOWED_HOSTS = ['*'].

Указываем параметры БД, которую мы создали выше:


DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',  # Database engine
        'NAME': 'netbox',         # Database name
        'USER': 'netbox',               # PostgreSQL username
        'PASSWORD': 'netboxpass',           # PostgreSQL password
        'HOST': 'localhost',      # Database server
        'PORT': '',               # Database port (leave blank for default)
        'CONN_MAX_AGE': 300,      # Max database connection age
    }
}
```

Далее при помощи команды `python3 -c "import secrets; print(secrets.token_urlsafe(64))"` гененрируем секретную строку (она нам понадобиться для создания api-токена, при помощи которого мы бдуем обращаться к Netbox API). Полученную после выполнения команды строку вставляем:

```
API_TOKEN_PEPPERS = {'string'}
```

Запускаем NetBox:

```
python3 manage.py migrate
python3 manage.py createsuperuser
python3 manage.py runserver 0.0.0.0:8000
```

### Настройка NetBox

В самом NetBox создаем и настраиваем следующие объекты: `Site`, `Devide Role`, `Device Type`, `Ip address`, `Primary Ip address`, `Interfaces` 

<img width="1846" height="966" alt="image" src="https://github.com/user-attachments/assets/2e699fb4-fde9-48f9-ab3f-cf56781e7faa" />

### Экспорт информации из NetBox

Плейбук предназначен для автоматизированного экспорта данных из системы NetBox в локальный JSON-файл. Выполнение производится на управляющем узле (localhost) без подключения к сетевым устройствам. Для доступа к данным используется API NetBox и токен аутентификации.

На первом этапе плейбук последовательно получает информацию об основных объектах инфраструктуры: площадках (Sites), ролях устройств (Device Roles), устройствах (Devices), интерфейсах (Interfaces) и IP-адресах (IP Addresses). Получение данных выполняется с помощью lookup-плагина netbox.netbox.nb_lookup, который обращается к API NetBox и возвращает объекты в формате Ansible.

После получения информации данные преобразуются в удобную структуру, содержащую только полезные значения объектов без служебных полей Ansible. Для этого используется фильтр map(attribute='value'), который извлекает содержимое каждого объекта NetBox.

На следующем этапе формируется единая структура netbox_export, включающая все собранные данные. В результате создаётся централизованный набор сведений о сетевой инфраструктуре, содержащий информацию об устройствах, их ролях, интерфейсах и назначенных IP-адресах.

На завершающем этапе плейбук сохраняет сформированную структуру в JSON-файл с использованием фильтра to_nice_json, обеспечивающего удобное форматирование выходных данных. Полученный файл может использоваться для резервного копирования информации из NetBox, анализа инфраструктуры, формирования отчётов или последующей автоматизации процессов управления сетью.

```

- name: Export NetBox data to file
  hosts: localhost
  gather_facts: false

  vars:
    netbox_api: "http://10.0.0.4:8000"
    netbox_token: "nbt_bjhKRqFNOLS3.mf9Qlc4bJ2fuU8FDiDLfKbaiDdj0DCLVkZEpiMXu"
    export_file: "/home/georgy/Documents/Comp-network-programming/LAB-2/exports/netbox/netbox_data.json"

  tasks:

    - name: Read sites from NetBox
      set_fact:
        nb_sites: "{{ query('netbox.netbox.nb_lookup', 'sites',
                            api_endpoint=netbox_api,
                            token=netbox_token) }}"

    - name: Read device roles from NetBox
      set_fact:
        nb_roles: "{{ query('netbox.netbox.nb_lookup', 'device-roles',
                            api_endpoint=netbox_api,
                            token=netbox_token) }}"

    - name: Read devices from NetBox
      set_fact:
        nb_devices: "{{ query('netbox.netbox.nb_lookup', 'devices',
                              api_endpoint=netbox_api,
                              token=netbox_token) }}"

    - name: Read interfaces from NetBox
      set_fact:
        nb_interfaces: "{{ query('netbox.netbox.nb_lookup', 'interfaces',
                                 api_endpoint=netbox_api,
                                 token=netbox_token) }}"

    - name: Read IP addresses from NetBox
      set_fact:
        nb_ips: "{{ query('netbox.netbox.nb_lookup', 'ip-addresses',
                          api_endpoint=netbox_api,
                          token=netbox_token) }}"

    - name: Build export structure
      set_fact:
        netbox_export:
          sites: "{{ nb_sites | map(attribute='value') | list }}"
          device_roles: "{{ nb_roles | map(attribute='value') | list }}"
          devices: "{{ nb_devices | map(attribute='value') | list }}"
          interfaces: "{{ nb_interfaces | map(attribute='value') | list }}"
          ip_addresses: "{{ nb_ips | map(attribute='value') | list }}"

    - name: Save NetBox export to JSON file
      copy:
        content: "{{ netbox_export | to_nice_json }}"
        dest: "{{ export_file }}"
```

<img width="1542" height="703" alt="image" src="https://github.com/user-attachments/assets/da6048ef-ed8e-490f-a455-e30a5eab8279" />

### Запись серийного номера в NetBox

Плейбук предназначен для автоматизированного сбора уникального идентификатора MikroTik CHR и его последующей записи в систему NetBox. Выполнение производится для группы маршрутизаторов routers с использованием подключения по SSH через Ansible и модуля community.routeros.command.

На первом этапе плейбук подключается к каждому маршрутизатору и выполняет команду /system license print, которая выводит информацию о лицензии RouterOS. Для виртуальных маршрутизаторов MikroTik CHR аппаратный серийный номер отсутствует, поэтому в качестве уникального идентификатора используется параметр system-id, генерируемый системой RouterOS.

Полученный вывод сохраняется в переменную license_out. Для контроля корректности работы дополнительно выводится исходный результат выполнения команды с помощью модуля debug.

Далее плейбук выполняет обработку полученных данных и извлекает значение поля system-id. Для повышения надёжности используется безопасный метод обработки строк без применения регулярных выражений, что исключает ошибки выполнения при отсутствии ожидаемого шаблона в выводе устройства. Извлечённое значение сохраняется в переменную router_serial.

После получения идентификатора выполняется обновление данных устройства в NetBox. С помощью модуля netbox.netbox.netbox_device плейбук подключается к API NetBox и записывает полученное значение в поле serial соответствующего устройства. Поиск устройства осуществляется по имени, совпадающему с именем узла в inventory.

В результате выполнения плейбука база данных NetBox автоматически синхронизируется с фактическим состоянием сетевых устройств, а поле серийного номера содержит актуальный уникальный идентификатор каждого маршрутизатора MikroTik CHR.

```

root@georgy-VirtualBox:/home/georgy/Documents/Comp-network-programming/LAB-2/playbooks# cat serial_netbox.yml 
- name: Collect CHR system-id and push to NetBox
  hosts: routers
  gather_facts: false

  vars:
    netbox_api: "http://10.0.0.4:8000"
    netbox_token: "nbt_bjhKRqFNOLS3.mf9Qlc4bJ2fuU8FDiDLfKbaiDdj0DCLVkZEpiMXu"

    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: community.routeros.routeros
    ansible_user: admin
    ansible_password: admin

  tasks:

    - name: Get license info
      community.routeros.command:
        commands:
          - /system license print
      register: license_out

    - name: Debug raw output
      debug:
        var: license_out.stdout

    - name: Extract system-id safely (NO regex group crash)
      set_fact:
        router_serial: >-
          {{
            (license_out.stdout[0].splitlines()
              | select('search', 'system-id')
              | first
              | default('system-id: unknown')
            ).split(':')[-1].strip()
          }}

    - name: Show extracted ID
      debug:
        msg: "system-id = {{ router_serial }}"

    - name: Push to NetBox
      delegate_to: localhost
      netbox.netbox.netbox_device:
        netbox_url: "{{ netbox_api }}"
        netbox_token: "{{ netbox_token }}"
        data:
          name: "{{ inventory_hostname }}"
          serial: "{{ router_serial }}"
        state: present
```

<img width="777" height="601" alt="image" src="https://github.com/user-attachments/assets/51bf431e-1d95-430e-b60d-c83a0572a8b1" />


### Конфигурация роутеров при помощи данных из NetBox

Данный плейбук предназначен для автоматической настройки маршрутизаторов MikroTik CHR с использованием информации, хранящейся в NetBox. NetBox выступает в роли Source of Truth и предоставляет актуальные параметры устройств через API.

При выполнении плейбук сначала обращается к NetBox и получает сведения об устройстве, имя которого совпадает с текущим узлом инвентаря Ansible. Из полученных данных извлекается Primary IPv4 Address, назначенный устройству в NetBox. Далее Ansible подключается к маршрутизатору по SSH с использованием подключения network_cli и выполняет настройку оборудования.

В процессе работы плейбук изменяет системное имя маршрутизатора на значение inventory_hostname, проверяет наличие интерфейса tunel (WireGuard), удаляет ранее назначенный адрес на данном интерфейсе и назначает новый IP-адрес, полученный из поля Primary IPv4 Address устройства в NetBox. Для удобства администрирования создаваемая запись сопровождается комментарием NetBox managed, что позволяет определить настройки, управляемые централизованно.

После применения конфигурации плейбук выполняет проверочные команды, отображающие текущее имя устройства и список IP-адресов. Полученный результат сохраняется локально в текстовый файл для последующего анализа и подтверждения корректности настройки.

Таким образом, плейбук демонстрирует практическое использование NetBox как централизованного источника данных и автоматическое применение параметров конфигурации на сетевых устройствах с помощью Ansible.

<img width="1469" height="497" alt="image" src="https://github.com/user-attachments/assets/7bbbbf53-e8d5-47a0-985c-f0c7f72d1f5d" />

## Вывод

В ходе выполнения лабораторной работы были изучены возможности интеграции системы управления сетевой инфраструктурой NetBox с системой автоматизации Ansible. Был развернут и настроен сервер NetBox, включающий PostgreSQL в качестве базы данных и Redis для обеспечения работы фоновых задач и кэширования. Для доступа к API был создан и настроен токен версии v2.

В рамках работы были разработаны и протестированы три Ansible-плейбука. Первый плейбук выполнял экспорт данных из NetBox и формировал локальный JSON-файл с информацией о площадках, устройствах, интерфейсах и IP-адресах. Второй плейбук осуществлял сбор уникального идентификатора MikroTik CHR (system-id) и автоматически записывал его в поле серийного номера соответствующего устройства в NetBox. Третий плейбук демонстрировал использование NetBox в качестве единого источника достоверных данных (Source of Truth): из NetBox автоматически получались параметры устройств, после чего выполнялась настройка маршрутизаторов, включая изменение имени устройства и назначение IP-адреса на интерфейс WireGuard.

В процессе выполнения работы были рассмотрены вопросы взаимодействия Ansible с API NetBox, использования коллекции netbox.netbox, работы с динамическими данными инвентаря, а также особенности автоматизации настройки устройств MikroTik RouterOS через SSH.

Полученные результаты подтвердили возможность централизованного хранения информации о сетевой инфраструктуре в NetBox и её последующего использования для автоматизированного управления сетевыми устройствами. Использование связки NetBox и Ansible позволяет существенно сократить объём ручных операций, повысить актуальность конфигурационных данных и снизить вероятность ошибок при администрировании сети.


