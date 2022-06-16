# Практика работы с SELinux

## Описание
  1. Запустить nginx на нестандартном порту 3-мя разными способами:
    переключатели setsebool;
    добавление нестандартного порта в имеющийся тип;
    формирование и установка модуля SELinux. К сдаче:
    README с описанием каждого решения (скриншоты и демонстрация приветствуются).

  2. Обеспечить работоспособность приложения при включенном selinux.
    развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
    выяснить причину неработоспособности механизма обновления зоны (см. README);
    предложить решение (или решения) для данной проблемы;
    выбрать одно из решений для реализации, предварительно обосновав выбор;
    реализовать выбранное решение и продемонстрировать его работоспособность. К сдаче:
    README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
    исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.

### Критерии оценки:
Статус "Принято" ставится при выполнении следующих условий:
для задания 1 описаны, реализованы и продемонстрированы все 3 способа решения;
для задания 2 описана причина неработоспособности механизма обновления зоны;
для задания 2 реализован и продемонстрирован один из способов решения. Опционально для выполнения:
для задания 2 предложено более одного способа решения;
для задания 2 обоснованно(!) выбран один из способов решения.

# Пошаговая инструкция выполнения домашнего задания:
В связи с недоступностью Vagrant cloud, бокс centos/7 был загружен локально на виртуальную машину с Ubuntu 20.04.3 как это и делалось ранее. Сам бокс дополнительно сохранил сохранил на диск: https://disk.yandex.ru/d/6FxLl6u6ILFMgQ
Далее добавил бокс:
```
vagrant box add centos7 CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box
```
Подготовил Vagrantfile:
```
# -*- mode: ruby -*-
# vim: set ft=ruby :
MACHINES = {
:selinux => {
:box_name => "centos7",
#:provision => "test.sh",
},
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = "selinux"
      box.vm.network "forwarded_port", guest: 4881, host: 4881
      box.vm.provider :virtualbox do |vb|
          vb.customize ["modifyvm", :id, "--memory", "512"]
          needsController = false
      end
      box.vm.provision "shell", inline: <<-SHELL
        #install epel-release
        yum install -y epel-release
        #install policycoreutils-python for audit2why
        yum install -y policycoreutils-python
        #install nginx
        yum install -y nginx
        #change nginx port
        sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
        sed -i 's/listen 80;/listen 4881;/'
        /etc/nginx/nginx.conf
        #disable SELinux
        #setenforce 0
        #start nginx
        systemctl start nginx
        systemctl status nginx
        #check nginx port
        ss -tlpn | grep 4881
      SHELL
     end
  end
end
```
Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.
Заходим на стенд и проверяем статус nginx:
```
sam@yarkozloff:/otus/selinux$ vagrant ssh
Last login: Thu Jun 16 21:02:58 2022 from 10.0.2.2
[vagrant@selinux ~]$ sudo systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[vagrant@selinux ~]$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-06-16 21:03:26 UTC; 11s ago
  Process: 2269 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2267 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Jun 16 21:03:26 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 21:03:26 selinux nginx[2269]: nginx: the configuration file /etc/nginx/nginx.conf synt...s ok
Jun 16 21:03:26 selinux nginx[2269]: nginx: [emerg] bind() to [::]:4881 failed (13: Permissio...ied)
Jun 16 21:03:26 selinux nginx[2269]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 16 21:03:26 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jun 16 21:03:26 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jun 16 21:03:26 selinux systemd[1]: Unit nginx.service entered failed state.
Jun 16 21:03:26 selinux systemd[1]: nginx.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
```
Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.
Заходим на сервер: vagrant ssh
Дальнейшие действия выполняются от пользователя root. Переходим в root пользователя: sudo -i

## 1. Запуск nginx на нестандартном порту 3-мя разными способами
Для начала проверим, что в ОС отключен файервол: 
```
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Конфигурация nginx настроена без ошибок:
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Режим работы SELinux:
```
[root@selinux ~]#  getenforce
Enforcing
```
Данный режим означает, что SELinux будет блокировать запрещенную активность.

### 1.1. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
Находим в логах информацию о блокировании порта:
```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1655409955.224:792): avc:  denied  { name_bind } for  pid=3094 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket
```
Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим информации о запрете. 
Но для этого нужна сама утилита, на стенде её не оказалось поэтому нужно доставить policycoreutils-python (добавил его установку в скрипте в Vagrantfile):
```
[root@selinux ~]# grep 1655409955.224:792 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1655409955.224:792): avc:  denied  { name_bind } for  pid=3094 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled.
Включим параметр nis_enabled и перезапустим nginx: setsebool -P nis_enabled on (немного подождать):
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-06-16 21:18:56 UTC; 5s ago
  Process: 2598 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2596 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2595 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2600 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2600 nginx: master process /usr/sbin/nginx
           └─2602 nginx: worker process

Jun 16 21:18:56 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 21:18:56 selinux nginx[2596]: nginx: the configuration file /etc/nginx/nginx.conf synt...s ok
Jun 16 21:18:56 selinux nginx[2596]: nginx: configuration file /etc/nginx/nginx.conf test is ...sful
Jun 16 21:18:56 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Проверяем curl-ом, что страница выдаётся:
```
[root@selinux ~]# curl -a http://localhost:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
       ...
       ...
```
Проверяем статус параметра командой:
```
[root@selinux ~]#  getsebool -a | grep nis_enabled
nis_enabled --> on
```
Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled. 
После отключения nis_enabled служба nginx снова не запустится.
```
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-06-16 21:22:32 UTC; 2s ago
  Process: 2598 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2623 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  ...
```
### 1.2. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип
Поиск имеющегося типа, для http трафика:
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t:
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
```
Запускаем nging, проверяем работу:
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# curl -a http://localhost:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  ...
```
Удаляем нестандартный порт из имеющегося типа и проверяем, что nginx не работает:
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-06-16 21:27:30 UTC; 8s ago
   ...
[root@selinux ~]# curl -a http://localhost:4881
curl: (7) Failed connect to localhost:4881; Connection refused
```

### 1.3. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:
```
[root@selinux ~]# curl -a http://localhost:4881
curl: (7) Failed connect to localhost:4881; Connection refused
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Audit2allow сообщил команду, с помощью которой можно применить сформированный модуль. Запускаем и проверяем nginx:
```
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# curl -a http://localhost:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
...
```
После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.
Просмотр всех установленных модулей: semodule -l
Для удаления модуля воспользуемся командой: semodule -r nginx

## 2. Обеспечить работоспособность приложения при включенном selinux.
Подготовка окружения. Потребуется Ansible и Git.
Git уже есть. Обновим Ansible, проверим версию:
```
sam@yarkozloff:/otus/selinux2$ ansible --version
ansible [core 2.12.4]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/sam/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/sam/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Mar 15 2022, 12:22:08) [GCC 9.4.0]
  jinja version = 2.10.1
  libyaml = True
```
Клонируем репозиторий:
```
sam@yarkozloff:/otus/selinux$ git clone https://github.com/mbfx/otus-linux-adm.git
Cloning into 'otus-linux-adm'...
remote: Enumerating objects: 542, done.
remote: Counting objects: 100% (440/440), done.
remote: Compressing objects: 100% (295/295), done.
remote: Total 542 (delta 118), reused 381 (delta 69), pack-reused 102
Receiving objects: 100% (542/542), 1.38 MiB | 549.00 KiB/s, done.
Resolving deltas: 100% (133/133), done.
```
Отредактируем Vagrantfile (чтобы использовать локальный бокс):
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos7"

  config.vm.provision "ansible" do |ansible|
    #ansible.verbose = "vvv"
    ansible.playbook = "provisioning/playbook.yml"
    ansible.become = "true"
  end

  config.vm.provider "virtualbox" do |v|
	  v.memory = 256
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "dns"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "dns"
    client.vm.hostname = "client"
  end

end
```
Переходим в нужный каталог, поднимаем стенды (пришлось подождать чуть дольше), проверяем:
```
sam@yarkozloff:/otus/selinux$ cd otus-linux-adm/selinux_dns_problems
sam@yarkozloff:/otus/selinux/otus-linux-adm/selinux_dns_problems$ vagrant up
sam@yarkozloff:/otus/selinux/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```
Подключаемся к клиенту:
```
sam@yarkozloff:/otus/selinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Thu Jun 16 22:07:12 2022 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################
```
Попробуем внести изменения в зону:
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> ^C[vagrant@client ~]$ ^C
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
Воспользуемся утилитой audit2why чтобы посмотреть в логах SELinux в чем проблема:
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]#
```
Тут мы видим, что на клиенте отсутствуют ошибки.
Подключимся к серверу ns01 и проверим логи SELinux:
```
sam@yarkozloff:~$ cd /otus/selinux/otus-linux-adm/selinux_dns_problems/
sam@yarkozloff:/otus/selinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Thu Jun 16 22:03:04 2022 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1655416985.410:1761): avc:  denied  { search } for  pid=5451 comm="isc-worker0000" name="net" dev="proc" ino=29143 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. 
Проверим данную проблему в каталоге /etc/named
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
etc_t - контекст безопасности неправильный. Конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них
распространялись правильные политики SELinux можно с помощью команды:
```
[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
```
Изменим тип контекста безопасности для каталога /etc/named:
```
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
named_zone_t - Изменения применились. Снова вносим изменения на клиенте:
```

```

