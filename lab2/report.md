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

