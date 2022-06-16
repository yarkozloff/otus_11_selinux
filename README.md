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

## Запуск nginx на нестандартном порту 3-мя разными способами
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

### 1. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
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
### 2. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип
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

### 3. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:
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

## 
