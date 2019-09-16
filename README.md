# kubernetes-tomcat-nginx-sample
This tutorial describes how to install kubernetes cluster with sample.war on tomcat and nginx-ingress on VirtualBox machines.

# Предварительные требования
1) Vagrant
2) VirtualBox
3) 40 Gb+ свободного места
4) ОС семейства Unix

# Настройка VagrantFile
Введите следующую команду:
```sh
$ ifconfig
```
В списке найдите параметр отвечающий за wlan. Для того, чтобы обеспечить внешний доступ к приложениям из кластера, необходимо, чтобы host и guest машины были в одном и том же ip диапазоне. В моем случае параметр wlp3s0 inet: 192.168.0.X. Откройте Vagrant файл и измените ip адреса виртуальных машин в соответствии с этим параметром.

# Запуск виртуальных машин
Перейдите в каталог Vagrant Files и выполните следующую команду:
```sh
$ vagrant up
```
Данная команда создаст 4 виртуальные машины в режиме "Сетевой мост". Во время разворачивания, VirtualBox будет запрашивать через какое устройство необходимо обеспечить выход в сеть - необходимо выбрать для каждой из машин параметр wlan (wlp3s0).

# Установка необходимых зависимостей
Выполните вход в виртуальную машину №4 следующей командой:
```sh
$ vagrant ssh n4
```
Необходимо установить следующие зависимости:
<ul> python-pip </ul>
<ul> sshpass </ul>
<ul> ansible </ul>

Выполните следующие команды в n4:
```sh
vagrant@n4 $ sudo apt-get -y --fix-missing install python-pip sshpass
vagrant@n4 $ sudo sudo pip install ansible
```
После установки зависимостей создайте каталог ansible:
```sh
vagrant@n4 $ mkdir ~/ansible
vagrant@n4 $ cd ansible
```
На машине хоста откройте файл hosts и измените его содержимое в соответствии с вашими ip адресами ВМ. В моём случае файл hosts выглядит следующим образом:
```
[all]
192.168.0.10 ansible_user=vagrant ansible_ssh_pass=vagrant
192.168.0.11 ansible_user=vagrant ansible_ssh_pass=vagrant
192.168.0.12 ansible_user=vagrant ansible_ssh_pass=vagrant
```
Скопируйте данный файл в каталог ansible на виртуальной машине n4. Также необходимо перенести в каталог ansible файлы vars.yml и playbook.yml  
Далее необходимо создать файл конфигурации ansible:
```sh
vagrant@n4 $ vi ~/.ansible.cfg
[defaults]
host_key_checking = False
```
