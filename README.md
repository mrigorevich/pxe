# Настройка PXE сервера для автоматической установки<br><br>
### Описание/Пошаговая инструкция выполнения домашнего задания:<br><br>
1.	Настроить загрузку по сети дистрибутива Ubuntu 24
2.	Установка должна проходить из HTTP-репозитория.
3.	Настроить автоматическую установку c помощью файла user-data
________________________________________________________________________
Поднимаем первый сервер из [Vagrantfile](https://github.com/mrigorevich/pxe/blob/main/Vagrantfile) <br><br>
На основном хосте:<br>
`vagrant up pxesrv`<br>
В провижине прописана установка нашего стека: dnsmasq в качестве DHCP, TFTP и PXE; nfs-kernel-server – NFS сервер; nginx – в качестве веб-сервера.<br><br>
На ВМ pxesrv:<br>
`ip a` # смотрим название интерфейса, на котором будет поднимать dhcp<br>
`nano /etc/dnsmasq.d/pxe.conf` # создаем конфиг для dnsmasq<br><br>
<img width="616" height="209" alt="image" src="https://github.com/user-attachments/assets/83298075-e44a-4e9a-8379-2c5c1f3a907c" /><br><br>

Скачиваем образы:<br>
`wget https://cdimage.ubuntu.com/ubuntu-server/noble/daily-live/current/noble-netboot-amd64.tar.gz` # для сетевой загрузки<br>
`wget https://cdimage.ubuntu.com/ubuntu-server/noble/daily-live/current/noble-live-server-amd64.iso` # образ с дистрибутивом<br><br>
Чтобы не грузить полный дистрибутив в оперативную память новой виртуалки (ресурсы у нас ограничены), распакуем образ:<br>
`mkdir -p /srv/images/ubuntu`<br>
`mount -o loop noble-live-server-amd64.iso /mnt`<br>
`cp -r /mnt/. /srv/images/ubuntu/`<br>
`chmod -R 755 /srv/images/ubuntu`<br>
`umount /mnt`<br>
Распакуем архив сетевой загрузки:<br>
`mkdir /srv/tftp`<br>
`tar -xzvf noble-netboot-amd64.tar.gz -C /srv/tftp/`<br>
Скопируем в полученный каталог два файла из дистрибутива:<br>
`cp /srv/images/ubuntu/casper/initrd /srv/tftp/amd64/`<br>
`cp /srv/images/ubuntu/casper/vmlinuz /srv/tftp/amd64/`<br>
`nano /srv/tftp/amd64/pxelinux.cfg/default` # пропишем параметры установки<br>
```
DEFAULT install
LABEL install
  KERNEL vmlinuz
  INITRD initrd
  APPEND ip=dhcp boot=casper netboot=nfs nfsroot=192.168.20.10:/srv/images/ubuntu nfsrootopts=vers=3 autoinstall ds=nocloud-net;s=http://192.168.20.10/ks/
```

Пропишем конфиг автоустановки:<br>
`mkdir /srv/ks`<br>
`touch meta-data` # файл можно оставить пустым<br>
`nano user-data`<br>
```
#cloud-config
autoinstall:
  version: 1
  interactive-sections: []
  apt:
    disable_components: []
    geoip: true
    preserve_sources_list: false
    primary:
      - arches: [amd64, i386]
        uri: http://ru.archive.ubuntu.com/ubuntu
      - arches: [default]
        uri: http://ports.ubuntu.com/ubuntu-ports
  drivers:
    install: false
  identity:
    hostname: linux
    password: "$6$sJgo6Hg5zXBwkkI8$btrEoWAb5FxKhajagWR49XM4EAOfO/Dr5bMrLOkGe3KkMYdsh7T3MU5mYwY2TIMJpVKckAwnZFs2ltUJ1abOZ."
    realname: otus
    username: otus
  kernel:
    package: linux-generic
  keyboard:
    layout: us
    toggle: null
    variant: ''
  locale: en_US.UTF-8
  storage:
    layout: 
```

Далее настроим NFS:<br>
`nano /etc/nfs.conf`<br>
Раскомментируем следующие строки:<br><br>
<img width="220" height="404" alt="image" src="https://github.com/user-attachments/assets/db125f4c-634b-4a37-a3e7-a5caf450e3b7" /><br><br>
`nano /etc/exports`<br><br>
<img width="679" height="66" alt="image" src="https://github.com/user-attachments/assets/b70cba74-f592-4031-8d05-c8f389d95729" /><br><br>
`exportfs -f`<br>
`exportfs -ra`<br>
`showmount -e` # проверка<br><br>
<img width="304" height="70" alt="image" src="https://github.com/user-attachments/assets/9bd80351-95a1-466b-99bf-2157d9e91cf2" /><br><br>
И настроим nginx.<br>
`nano /etc/nginx/sites-available/ks-server.conf`<br><br>
<img width="459" height="384" alt="image" src="https://github.com/user-attachments/assets/6eda0930-1331-4ec0-8eea-d32e82ba4515" /><br><br>
`ln -s /etc/nginx/sites-available/ks-server.conf /etc/nginx/sites-enabled/` # включаем сайт<br>
`rm /etc/nginx/sites-enabled/default` # убираем дефолтный сайт<br><br>
Сохраняем и перезапускаем сервисы:<br>
`systemctl restart nginx`<br>
`systemctl restart nfs-kernel-server`<br>
`systemctl restart dnsmasq`<br>
`systemctl restart rpcbind`<br><br>
Включаем клиентскую машину.<br><br>
В Vagrantfile в конфигураторе для клиентской ВМ указаны доп.условия и изменены характеристики для того, чтобы установка из дистрибутива Ubuntu 24 смогла успешно пройти:<br><br>
<img width="974" height="166" alt="image" src="https://github.com/user-attachments/assets/2721fc0d-920d-4c62-8fbd-aa30b86a4943" /><br><br>
После завершения установки вернемся к этому фрагменту Vagrantfile.<br><br>
На хосте:<br>
`vagrant up pxecl`<br>
После включения ВМ в VirtualBox, открываем ее консоль и наблюдаем за процессом<br><br>
<img width="580" height="368" alt="image" src="https://github.com/user-attachments/assets/32db2eea-3c39-40c6-af8b-93428c35fef1" /><br><br>
Загрузка установщика:<br><br>
<img width="635" height="321" alt="image" src="https://github.com/user-attachments/assets/818257d1-65cd-4aab-ba25-a7b17a7bd228" /><br><br>
Автоустановка:<br><br>
<img width="658" height="925" alt="image" src="https://github.com/user-attachments/assets/1362d668-24fd-41bf-ad54-5ebfd19eacae" /><br><br>
По окончании установки выключаем ВМ в интерфейсе VirtualBox.<br><br>
Возвращаемся к Vagranfile и комментируем строки, которые были нужны для установки, в порядке загрузки меняем net на disk:<br><br>
<img width="562" height="154" alt="image" src="https://github.com/user-attachments/assets/3ce56ce3-0d6d-4f6d-8206-27be8abf650b" /><br><br>
На хосте запускаем ВМ:<br>
`vagrant up pxecl`<br><br>
После этого в консоли VirtualBox проверяем нашу ВМ:<br><br>
<img width="692" height="992" alt="image" src="https://github.com/user-attachments/assets/e3ecba43-14fb-4856-b46f-b6857902ea51" /><br><br>
Залогинились с заданными учетными данными, автоустановка корректно отработала.
