# Практика работы с SELinux
## Описание
Запустить nginx на нестандартном порту 3-мя разными способами:
переключатели setsebool;
добавление нестандартного порта в имеющийся тип;
формирование и установка модуля SELinux. К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются).
Обеспечить работоспособность приложения при включенном selinux.
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

## Пошаговая инструкция выполнения домашнего задания:
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
