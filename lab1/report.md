# Отчет по лабораторной работе №1 - "Установка CHR и Ansible, настройка VPN".

## Шапка отчета

* University: [ITMO University](https://itmo.ru/ru/)
* Faculty: [ФПиН](https://fpin.itmo.ru/ru)
* Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
* Year: 2025/2026
* Group: K3322
* Author: Titov Georgy Konstantinovich
* Lab: Lab1
* Date of create: 22.05.2026
* Date of finished: 22.05.2026

## Описание

Данная работа предусматривает обучение развертыванию виртуальных машин (VM) и системы контроля конфигураций Ansible а также организации собственных VPN серверов.

## Цель работы

Целью данной работы является развертывание виртуальной машины на базе платформы Microsoft Azure с установленной системой контроля конфигураций Ansible и установка CHR в VirtualBox

## Задание

Вам необходимо развернуть виртуальную машину с помощью Microsoft Azure в режиме студенческой подписки.
Если не получается в Microsoft Azure, можете выбрать любого бесплатного облачного провайдера

В бесплатном режиме Microsoft Azure предлагает для виртуальных машин только Ubuntu 16.4, нам нужна Ubuntu 18.+ поэтому необходимо обновить операционную систему. Сделать это можно с помощью данных команд:

```
sudo apt update & sudo apt upgrade
sudo do-release-upgrade
```

Теперь необходимо установить python3 и Ansible:

```
sudo apt install python3-pip
ls -la /usr/bin/python3.6
sudo pip3 install ansible
ansible --version
```

Далее вам необходимо на вашем компьютере установить VirtualBox а на него CHR (RouterOS).
> В интернете есть инструкции по установке CHR на VirtualBox, одна из этих инструкций вам поможет.

После этого вам необходимо создать свой Wireguard/OpenVPN/L2TP сервер для организации VPN туннеля между вашим сервером автоматизации где был установлена система контроля конфигураций Ansible и вашим локальным CHR.
> В интернете есть инструкции по установке Wireguard/OpenVPN/L2TP сервер для Ubuntu 18, одна из этих инструкций вам поможет.

После всех манипуляций вам необходимо будет поднять VPN туннель между вашим VPN сервером на Ubuntu 18 и VPN клиентом на RouterOS (CHR)
> Что искать? («Название вашего протокола» клиент RouterOS).

Результаты лабораторной работы
В результате лабораторной работы у вас должна получиться схема связи следующего вида:

<img width="401" height="469" alt="image" src="https://github.com/user-attachments/assets/6ef9f960-4824-413c-bf47-60e4fd4b1d1d" />


## Выполнение работы

В качестве хоста VPS был выбран Хост-провайдер [HOSTKEY](https://hostkey.com/)

### Конфигурация сервера

<img width="1280" height="151" alt="image" src="https://github.com/user-attachments/assets/14d0ae32-25da-4473-969d-1e35efb32fd9" />

Далее в задании от нас требовалось установить Python и Ansible. У меня они уже были предустановленны:

<img width="864" height="132" alt="image" src="https://github.com/user-attachments/assets/b1961a36-054f-423e-8c52-a74129923d1a" />

<img width="1280" height="201" alt="image" src="https://github.com/user-attachments/assets/2a7b3768-4cd8-4136-89d3-385cdb1ea066" />

### Установка CHR

Скачиваем VDI образ с офф. сайта [Mikrotik](https://mikrotik.com/download?architecture=x86). Его используем в качестве диска в VirtualBox.  
Также в сетевых настройках выставляем Bridged Adapter на нужный интерфейс ноутбука. Это позволит подключить роутер к сети

<img width="920" height="531" alt="image" src="https://github.com/user-attachments/assets/a03ea70a-a959-4e88-8307-a5e4aef6fab0" />

Далее запускаем виртуальную машину, логинимся в системе `admin:admin`. И проверяем что роутер подключен к сети:

<img width="750" height="504" alt="image" src="https://github.com/user-attachments/assets/4ac6c6db-6058-418d-bc6b-04af0231b2f8" />

### Настройка WireGuard на VPS и VM

На VPS генерируем ключи командой:

```
wg genkey | tee privatekey | wg pubkey > publickey
```

Далее пропишем конфиг для Wireguard service:

```
[Interface]
Address = 10.220.220.1/24
ListenPort = 8888
PrivateKey = <private-key>

[Peer]
# CHR
PublicKey = <pub-key Wireguard VM>
AllowedIPs = 10.220.220.2/32
```

Далее перейдем к насторйке VM.  
Создаем виртуальный интерфейс (ключи для wiregurd сгенерируются при создании интерфейса автоматически):

```
/interface/wireguard
add listen-port=8888 name=tunel-1
```

Далее на VM назначем пир. В качестве пира будет выступать наш VPS:

```
/interface/wireguard/peers
add allowed-address=10.220.220.1/24 endpoint-address=<VPS-ip> endpoint-port=8888 interface=portal public-key="<pub-key Wireguard VPS>"
```

Назначем IP на WireGuard-интерфейс на VM и добавление маршрута:

```
/ip/address
add address=10.220.220.2/32 interface=portal
/ip/route
add dst-address=10.220.220.0/24 gateway=portal
```

### Результат

Попробуем пропинговать наш Wireguard сервер:

<img width="798" height="159" alt="image" src="https://github.com/user-attachments/assets/2fbcd31c-9bf0-494e-b71d-cda8d00f420c" />

<img width="1280" height="849" alt="29cdb8df-d9d6-429f-a150-e0cfbef2011d" src="https://github.com/user-attachments/assets/fe1dcb89-2d0d-4f47-ad62-9c9df59dd2eb" />

## Вывод

В ходе лабораторной работы была развернута облачная виртуальная машина с Ubuntu, установлены Python и Ansible, а также развернут локальный CHR в VirtualBox. Настроен VPN-туннель на базе WireGuard между VPS и CHR, включая обмен ключами, настройку интерфейсов, IP-адресацию и маршрутизацию. В результате обеспечена стабильная IP-связность между узлами через защищённый туннель, что подтвердило работоспособность конфигурации.
