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
В связи с недоступностью Vagrant cloud, бокс centos/7 был загружен локально на виртуальную машину с Ubuntu 20.04.3 как это и делалось ранее. Сам бокс дополнительно сохранил в свой локальный репозиторий (который создал на прошлом дз на другом сервере):