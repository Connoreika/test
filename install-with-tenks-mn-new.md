Установка виртуального региона Kayobe с несколькими сетямт, используя tenks
==========================================================================

## Описание

> ИНСТРУКЦИЯ НЕ ЗАКОНЧЕНА

Эта инструкция содержит список действий для поднятия виртуального региона на 1 сервере с помощью вложенной виртуализации. Физический сервер должен настроиться под libvirt-гипервизор с шестью виртуалками:

- seed - виртуалка с bifrost и docker-registry. Единственная виртуалка, которая поднимается скриптами kayobe(не через tenks)
- ctrl[0-2] - три контроллера нового региона
- cmp[0-1] - две компьют ноды

Особенности региона:

- Локальный docker-regitry
- Три сети для сервисов overcloud
- Отдельная сеть для виртуалок в инстансах (OVS)
- Использование tenks для поднятия виртуалок ctrl и cmp. Для этих виртуалок tenks создаёт псевдо-IPMI на vbmc, креды которых автоматически прописываются в bifrost.

## Подготовка стенда

### Пререквизиты
- OS Centos 8
- 32CPU-64RAM-300Gb HDD
- 3 сетевых интерфейса
- **Все доступное дисковое пространство добавить в корень.**

Предварительно необходимо выполненить следующие действия:

- Стать рутом
```
sudo -i
```
- Отключить firewall
```
systemctl is-enabled firewalld && sudo systemctl stop firewalld && sudo systemctl disable firewalld
```
- Отключить selinux
```
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0  - не работает в centos8, нужно грузить ВМ!
```
- Перезагрузить ВМ, чтобы изменения вступили в силу
```
reboot
```
- Стать рутом
```
sudo -i
```

- Обновить yum-пакеты и установить сопутствующие инструменты

```
dnf update -y
dnf install -y epel-release git tmux
dnf install -y screen
```

## Подготовка файлов конфигураций

Для последующей работы потребуются три репозитория. kayobe - непосредственно скрипты kayobe, kayobe-config - базовый набор файлов для конфигурации будущего региона, tenks - набор плейбуков для поднятия виртуальных контроллеров и инстансов.

```
rm -rf /opt/kayobe

git clone -b stable/ussuri https://github.com/openstack/kayobe.git /opt/kayobe/src/kayobe
#git clone -b stable/ussuri https://github.com/openstack/kayobe-config.git /opt/kayobe/src/kayobe-config
git clone -b vtb/ussuri-tenks-default https://bitbucket.region.vtb.ru/scm/rock/kayobe-config.git /opt/kayobe/src/kayobe-config
git clone -b master https://github.com/openstack/tenks.git /opt/kayobe/src/tenks
```

## Работа с Seed-hypervisor и Seed

Теперь можно запускать скрипты настройки физической машины и seed-виртуалки.

```
# Поднимаем временный бридж, который в ходе работы плейбуков kayobe control host bootstrap пропишется в системе
if ! ip l show brmgmt >/dev/null 2>&1; then
  ip l add brmgmt type bridge
  ip l set brmgmt up
  ip a add 192.168.33.4/24 dev brmgmt
fi
if ! ip l show brcloud >/dev/null 2>&1; then
  ip l add brcloud type bridge
  ip l set brcloud up
  ip a add 192.168.44.4/24 dev brcloud
fi
if ! ip l show brpub >/dev/null 2>&1; then
  ip l add brpub type bridge
  ip l set brpub up
  ip a add 192.168.45.4/24 dev brpub
fi

iface=$(ip -o r get 1 | sed -n 's/.*dev \(.*\+\) src.*/\1/p')
iptables -A POSTROUTING -t nat -o $iface -j MASQUERADE
sysctl -w net.ipv4.conf.all.forwarding=1

cd /opt/kayobe/src/kayobe

#Создадим виртуалэнв с кайобой и начнем его использовать.
virtualenv -v -p python3 --clear --system-site-packages /opt/kayobe/venvs/kayobe
. /opt/kayobe/venvs/kayobe/bin/activate

pip install --no-cache-dir --no-color /opt/kayobe/src/kayobe
. /opt/kayobe/src/kayobe-config/kayobe-env

#Забутстрапим хост как контролхост
kayobe -vvvv control host bootstrap 2>&1 | tee ~/ControlHostBootstrap.log
egrep "(unreachable|failed)=[1-9]" ~/ControlHostBootstrap.log

#Сконфигурируем хост как гипервизор для виртуалки сида
kayobe -vvvv seed hypervisor host configure 2>&1 | tee ~/SeedHypervisorHostConfigure.log
egrep "(unreachable|failed)=[1-9]" ~/SeedHypervisorHostConfigure.log

# Удаляем стандартную сеть libvirt
virsh net-destroy default
virsh net-undefine default

# Создадим и настроим seed-виртуалку.
kayobe -vvvv seed vm provision 2>&1 | tee ~/SeedVmProvision.log
egrep "(unreachable|failed)=[1-9]" ~/SeedVmProvision.log

kayobe -vvvv seed host configure 2>&1 | tee ~/SeedHostConfigure.log
egrep "(unreachable|failed)=[1-9]" ~/SeedHostConfigure.log
```

### Загрузка docker-образов в локальный docker-regitry

```
images="kolla/centos-source-bifrost-deploy \
kolla/centos-source-kolla-toolbox \
kolla/centos-source-haproxy \
kolla/centos-source-mariadb \
kolla/centos-source-mariadb-clustercheck \
kolla/centos-source-fluentd \
kolla/centos-source-cron \
kolla/centos-source-keepalived \
kolla/centos-source-neutron-server \
kolla/centos-source-neutron-l3-agent \
kolla/centos-source-neutron-metadata-agent \
kolla/centos-source-neutron-openvswitch-agent \
kolla/centos-source-neutron-dhcp-agent \
kolla/centos-source-glance-api \
kolla/centos-source-nova-compute \
kolla/centos-source-keystone-fernet \
kolla/centos-source-keystone-ssh \
kolla/centos-source-keystone \
kolla/centos-source-nova-api \
kolla/centos-source-nova-conductor \
kolla/centos-source-nova-ssh \
kolla/centos-source-nova-novncproxy \
kolla/centos-source-nova-scheduler \
kolla/centos-source-placement-api \
kolla/centos-source-openvswitch-vswitchd \
kolla/centos-source-openvswitch-db-server \
kolla/centos-source-nova-libvirt \
kolla/centos-source-memcached \
kolla/centos-source-rabbitmq \
kolla/centos-source-chrony \
kolla/centos-source-heat-api \
kolla/centos-source-heat-api-cfn \
kolla/centos-source-heat-engine \
kolla/centos-source-horizon \
kolla/centos-source-kibana \
kolla/centos-source-elasticsearch \
kolla/centos-source-cinder-scheduler \
kolla/centos-source-cinder-api \
kolla/centos-source-cinder-volume \
kolla/centos-source-cinder-backup \
kolla/centos-source-tempest \
kolla/centos-source-iscsid \
kolla/centos-source-ironic-pxe \
kolla/centos-source-ironic-api \
kolla/centos-source-ironic-conductor \
kolla/centos-source-ironic-inspector \
kolla/centos-source-dnsmasq \
kolla/centos-source-nova-compute-ironic \
kolla/centos-source-ironic-neutron-agent \
kolla/centos-source-swift-base \
kolla/centos-source-swift-proxy-server \
kolla/centos-source-swift-account \
kolla/centos-source-swift-rsyncd \
kolla/centos-source-swift-container \
kolla/centos-source-swift-object-expirer \
kolla/centos-source-swift-object"

for image in ${images[@]}; do
    ssh stack@192.168.33.5 sudo docker pull $image:ussuri
    ssh stack@192.168.33.5 sudo docker tag $image:ussuri 192.168.33.5:4000/$image:ussuri
    ssh stack@192.168.33.5 sudo docker push 192.168.33.5:4000/$image:ussuri
done
```

### Деплоим контейнеры seed

```
kayobe -vv seed service deploy 2>&1 | tee ~/SeedServiceDeploy.log
egrep "(unreachable|failed)=[1-9]" ~/SeedServiceDeploy.log
deactivate
```

### Добавляем правило, чтобы docker registry был доступен

```
ssh stack@192.168.33.5 "sudo docker exec bifrost_deploy firewall-cmd --add-port 4000/tcp"
```

## Настройка и запуск Tenks

### Установка OVS

> Openvswitch необходим для работы tenks.

```
dnf install -y centos-release-openstack-ussuri
dnf install -y openvswitch
systemctl enable openvswitch
systemctl start openvswitch
```

### Запуск Tenks

```
cd /opt/kayobe/src/kayobe
export TENKS_CONFIG_PATH=/opt/kayobe/src/kayobe-config/etc/kayobe/tenks.yml
export KAYOBE_CONFIG_SOURCE_PATH=/opt/kayobe/src/kayobe-config
export KAYOBE_VENV_PATH=/opt/kayobe/venvs/kayobe
/opt/kayobe/src/kayobe/dev/tenks-deploy-overcloud.sh /opt/kayobe/src/tenks
```

По итогу, можно проверить наличие виртуалок и тестовых сред

```
virsh list --all # Должно быть семь виртуалок, среди которых включённая только виртуалка seed.

source /root/tenks-venv/bin/activate
vbmc list # Должны появиться шесть пунктов с одинаковым адресом и статусом running. Статус running не должен смущать, так как это статус псевдо-IPMI, а не статус самих виртуалок.
deactivate
```

## Работа с Overcloud

Теперь, когда виртуалки созданы, можно на них разворачивать новый регион.

```
. /opt/kayobe/venvs/kayobe/bin/activate
. /opt/kayobe/src/kayobe-config/kayobe-env

kayobe -vvv overcloud inventory discover && \
kayobe -vvv overcloud hardware inspect && \
kayobe -vvv overcloud provision && \
kayobe -vvv overcloud host configure && \
kayobe -vvv overcloud container image pull && \
kayobe -vvv overcloud service deploy && \
. /opt/kayobe/src/kayobe-config/etc/kolla/public-openrc.sh && \
kayobe -vvv overcloud post configure && \
kayobe -vvv overcloud host command run --command "iptables -P FORWARD ACCEPT" --become --limit controllers

deactivate
```

## Создание virtualenv с openstack-cli

```
virtualenv /opt/kayobe/venvs/os-venv
source /opt/kayobe/venvs/os-venv/bin/activate
pip install -U pip
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/ussuri
. /opt/kayobe/src/kayobe-config/etc/kolla/public-openrc.sh
openstack endpoint list

## У kolla-ansible есть скрипт, в котором перечисленны команды для приготовления необходимых условий, с помощью которых можно поднять тестовую виртуалку на одном из инстансов.
/opt/kayobe/src/kolla-ansible/tools/init-runonce

deactivate
```%
