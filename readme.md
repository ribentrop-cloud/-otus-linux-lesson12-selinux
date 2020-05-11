# Описание решения
### 1. Nginx на нестандартном порту:
__Задание__: _Запустить nginx на нестандартном порту 3-мя разными способами_.
 __Пояснение__: для сокращения времени на развертывание демонстрационной среды (и из-за лени тоже) я взял за основу ДЗ по __systemd__, результатом которой были два работающих на разных портах сервиса __Apache__, а не __nginx__. Если принципиально __nginx__, готов переделать.

В _vm-selinux_ подготовлена следующая конфигурация сервисов __Apache__ для запуска на разных портах:
 
service name|config file|port|что сделано в SElinux, чтобы работало
---|---|---|---
httpd@first|/etc/httpd/conf/first.conf|9009|добавлен нестандартный порт
httpd@second|/etc/httpd/conf/second.conf|10010|сформирован и установлен модуль SELinux
httpd@third|/etc/httpd/conf/third.conf|11011|установлен переключатель setsebool


1. Для запуска __Apache__ (httpd@first) на порту _9009_ через "_добавление нестандартного порта в имеющийся тип_" выполнено следующее:

1.1. Добавлен порт _9009_ в разрешенные порты для _http_:
```
semanage port -a -t http_port_t -p tcp 9009
```
1.2. Выполнена проверка и успешно запущен __Apache__ (httpd@first):
```
[root@vm-selinux vagrant]# semanage port -l | grep ^http_port_t
http_port_t                    tcp      9009, 80, 81, 443, 488, 8008, 8009, 8443, 9000
...
[root@vm-selinux vagrant]# systemctl start httpd@first
[root@vm-selinux vagrant]# systemctl status  httpd@first
...
 Active: active (running) since Mon 2020-05-11 10:10:43 UTC; 3s ago
```

2. Для запуска _nginx_ на порту _XX_ через "_формирование и установка модуля SELinux_" ...
2.1. Безуспешно пытаемся запустить __Apache__ (httpd@third) и после формируем политику с помощью _audit2allow_:
```
[root@vm-selinux vagrant]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log
```
2.2. Применяем политику 
```
[root@vm-selinux vagrant] semodule -i httpd_add.pp
```
3.3. Запускаем __Apache__ (httpd@third). Теперь успешно:
```
[root@vm-selinux vagrant]# systemctl start httpd@third
[root@vm-selinux vagrant]# systemctl status  httpd@third
...
 Active: active (running) since Mon 2020-05-11 10:10:43 UTC; 3s ago
```
3.4 Если посмотреть на политику, то, якобы ее действия аналогичны _using the boolean 'nis_enabled'_, однако исходя из строки _"allow httpd_t..."_ видно, что скоуп "послаблений" значительно меньше. А именно затрагивается только "_unreserved_port_" для _httpd_t_:
```
[root@vm-selinux vagrant]# cat httpd_add.te
...
#!!!! This avc can be allowed using the boolean 'nis_enabled'
allow httpd_t unreserved_port_t:tcp_socket name_bind;
...
```
3. Для запуска _nginx_ на порту _10010_ c использованием __setsebool__.

1.1 При запуске __Apache__ (httpd@second)  audit.log перенаправлен на _audit2why_ для получение рекомендаций. Получаем рекомендацию изменить значения _boolean nis_enabled_:
```
# audit2why < /var/log/audit/audit.log
...
Was caused by:
The boolean nis_enabled was set incorrectly.
...
Allow access by executing:
# setsebool -P nis_enabled 1
```

1.2 Можно посмотреть, на что повлияет установка _boolean nis_enabled_ в __true__:
```
sesearch -b nis_enabled -A
```
Как видим, много на что. И это плохо, т.к. слишком много "послаблений" ради _httpd_ на нестандартном порту.
1.3 Тем не менее, устанавливаем _nis_enabled _ и запускаем сервис:
```
[root@vm-selinux vagrant]# setsebool -P nis_enabled 1
...
[root@vm-selinux vagrant]# systemctl start httpd@second
[root@vm-selinux vagrant]# systemctl status  httpd@second
...
 Active: active (running) since Mon 2020-05-11 10:10:43 UTC; 3s ago
```


__Проверка:__
Стенд реализован с помощью Vagrant. Вся изложенная выше последовательность установки __Apache__ и изменений в __SElinux__ отражена в _provision_-скрипте _make-multiple-httpd.sh_.
1. Запускаем виртуальную машину, заходим и переходим в _root_:
```
# vagrant up
# vagrant ssh
# sudo -s
```
2. Проверяем, что все __Apache__ (httpd@*) бегут на нестандартных портах:
```
netstat -tulpn | grep httpd
```
3. Проверяем, что к типу _http_port_t_ добавлен нестандартный порт (для httpd@first):
```
semanage port -l | grep ^http_port_t
```
4. Проверяем применение политики разрешения порта (для httpd@second):
```
 checkmodule -M -m -o httpd_add.mod httpd_add.te
```
Саму политику можно посмотреть тут:
```
cat /home/vagrant/httpd_add.te
```
4. Проверяем состояние _boolean nis_enabled_ (для httpd@third):
```
semanage boolean -l | grep nis_
```

