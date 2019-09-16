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
Запустите playbook.yml с помощью такой команды
```sh
vagrant@n4:~/ansible$ ansible-playbook -i hosts playbook.yml
```
Эта команда установит все необходимые зависимости на виртуальные машины n1, n2 и n3.

# Создание кластера

Выполните вход в виртуальную машину n1 (из хоста):
```sh
$ vagrant ssh n1
```
Выполним инициализацию кластера следующей командой:
```sh
vagrant@n1 $ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address <your-n1-machine-ip>
```
В моем случае команда выглядит следующем образом:
```sh
vagrant@n1 $ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address 192.168.0.10
```
Данная команда объявляет ВМ n1 <b>мастер</b> нодой.

После этого необходимо выполнить следующие команды
```sh
  vagrant@n1 $ mkdir -p $HOME/.kube
  vagrant@n1 $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  vagrant@n1 $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Настройка Pod сети.
В качестве сети для подов будем использовать flannel.  
Выполните команду в мастер ноде:
```sh
vagrant@n1 $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Спустя пару минут проверьте состояние вашей ноды:
```sh
vagrant@n1 $ kubectl get nodes
```
Вы должны увидеть, что статус мастер ноды - Ready

## Подключение остальных нод к кластеру
В случае успешной инициализации мастер ноды вы должны были увидеть сообщение вида
```sh
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join <ip> --token ...etc
```
Скопируйте эту команду и выполните ее на виртуальных машинах n2 и n3 (Не забудьте использовать sudo). Эта команда присоединит данные ноды к кластеру.  
После этого убедитесь, что ноды успешны добавлены и их статус Ready (процесс может занимать несколько минут):
```sh
vagrant@n1 $ kubectl get nodes
```
# Установка Nginx-ingress
Kubernetes не поддерживает стандартнные (облачные) балансироващики нагрузки (Load Balancer) для "bare metal" кластеров (наш случай), поэтому в качестве него мы будем использовать nginx-ingress.  
Прежде всего необходимо установить MetallB:
```sh
vagrant@n1 $ kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
```
Далее выполните следующие команды:
```sh
vagrant@n1 $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
vagrant@n1 $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
```

Проверим состояние установки следующей командой:
```sh
vagrant@n1 $ kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```
После того, как состояние станет "Ready" завершите процесс комбинацией CTRL+C

### Настройка nginx-ingress
Создайте каталог manifests
```sh
vagrant@n1 $ mkdir manifests
vagrant@n1 $ cd manifests/
```
Далее необходимо создать ConfigMap:
```yaml
vagrant@n1~/manifests $ vi ConfigMap.yml

apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.11-192.168.0.12
```
Убедитесь, что в addresses установлены ip адреса ВМ <b>n2</b> и <b>n3</b>  
Далее, выполните команду
```
vagrant@n1~/manifests $ kubectl create -f ConfigMap.yml
```

Проверим состояние сервиса командой:
```sh
vagrant@n1 $ kubectl -n ingress-nginx get svc
```
Вы должны увидеть вывод похожим на этот:
```sh
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.109.167.116   <none>        80:31841/TCP,443:30470/TCP   2m37s
```
В данном случае порт 80 - это внутренний порт, а порт 31841 - внешний. 
Для того, чтобы узнать на какой ноде установлен контроллер выполните команду:
```sh
vagrant@n1 $ kubectl get pods --all-namespaces -o wide
```
В моем случае ingress установился на ноду <b>n3</b>. Проверим пингуется ли данный порт с машины хоста:
```sh
$ ping 192.168.0.12 -p 31841
```
В случае успеха, должны посылаться пакеты.

# Установка приложений
Скопируйте файлы Deployment.yaml, Service.yaml и Ingress.yaml в папку /manifests на мастер ноду.

Далее, выполните следующую команду:
```sh
vagrant@n1~/manifests $ kubectl create -f Deployment.yml
```
Через пару минут проверьте состояние подов командой:
```sg
vagrant@n1~/manifests $ kubectl get pods
```
Вы должны увидеть состояние "Ready".  
Создадим также Service и Ingress
```sh
vagrant@n1~/manifests $ kubectl create -f Service.yml
vagrant@n1~/manifests $ kubectl create -f Ingress.yml
```
Готово! Осталось выполнить запрос в браузере с хоста:
https://192.168.0.12:30470/ (Обратите внимание на порт: вы должны подставить своё значение на который перенаправляется порт 443 сервиса ingress-nginx см. "Настройка nginx-ingress" )

В случае успеха, вы должны увидеть сообщение приветствие от tomcat.



