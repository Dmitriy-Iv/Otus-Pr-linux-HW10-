# **Введение**

В данном домашнем задании нам необходимо получить практический опыт работы с SELinux.

---

# **Подготовка и запуск окружения** 

1. Создаём файл Vagrantfile и вносим в него содержимое из методички:
```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
    :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",
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
                vb.customize ["modifyvm", :id, "--memory", "1024"]
                needsController = false
        end
        box.vm.provision "shell", inline: <<-SHELL
            #install epel-release
            yum install -y epel-release
            #install nginx
            yum install -y nginx
            #change nginx port
            sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
            sed -i 's/listen 80;/listen 4881;/' /etc/nginx/nginx.conf
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

2. После этого запускаем наш Vagrantfile.
```
dima@Test-Ubuntu-1:~/otus/my-hw11$ vagrant up
```

3. При разворачивании нашей vm видим следующие ошибки:
![alt text](/screenshots/hw11-1.PNG?raw=true "Screenshot1")

4. Далее заходим в vm и пытаемся исправить ошибку и запустить nginx.
```
dima@Test-Ubuntu-1:~/otus/my-hw11$ vagrant ssh
[vagrant@selinux ~]$ sudo -i
```
---

# **Запуск nginx на нестандартном порту 3-мя разными способами** 

### **Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool**

1. Проверяем, что firewalld отключен, конфигурация nginx не имеет ошибок, и статус самой системы SELinux.
![alt text](/screenshots/hw11-2.PNG?raw=true "Screenshot2") 

2. Для дальнейшего удобного анализа логов аудита устанавляиваем утилиту audit2why, которая входит в состав пакета policycoreutils-python.
```
[root@selinux ~]# yum -y install policycoreutils-python
```

3. Находим в логе /var/log/audit/audit.log информацию о блокировке порта и передаём время записи на вход утилите audit2why, которая нам покажет почему идёт блокировка и как её исправить.
![alt text](/screenshots/hw11-3.PNG?raw=true "Screenshot3") 

4. Как и предлагает нам утилита, включаем nis_enabled, перезагружаем nginx, проверяем что стартовая страница открывается на порту 4881.
![alt text](/screenshots/hw11-4.PNG?raw=true "Screenshot4") 

5. Возвращаем значение параметра nis_enabled в off.
```
[root@selinux ~]# setsebool -P nis_enabled off
```

### **Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип** 

1. Проверяем, какие порты у нас открыты для трафика http и убеждаемся, что нашего порта 4881 там нет.
![alt text](/screenshots/hw11-5.PNG?raw=true "Screenshot5") 

2. Добавляем в SELinux с помощью semanage port 4881, перезагружаем nginx, проверяем что стартовая страница открывается на порту 4881.
![alt text](/screenshots/hw11-6.PNG?raw=true "Screenshot6")

3. Удаляем порт 4881, и возвращаем nginx в нерабочее состояние.
![alt text](/screenshots/hw11-7.PNG?raw=true "Screenshot7")

### **Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux**

1. Проверяем log и с помощью утилиты audit2allow находим, как нам сделать модуль свой модуль для nginx на нестандартном порту.
![alt text](/screenshots/hw11-8.PNG?raw=true "Screenshot8")

2. Выполняем команду, с помощью которой применяем предложенный нам модуль, стартуем nginx и проверяем его работоспособность.
![alt text](/screenshots/hw11-9.PNG?raw=true "Screenshot9")

3. Отключаем модуль и возвращаем nginx в нерабочее состояние.
![alt text](/screenshots/hw11-10.PNG?raw=true "Screenshot10")

---

# **Обеспечение работоспособности приложения при включенном SELinux** 

### **Вариант 1 - Изменим тип контекста безопасности для каталога**

1. Клонируем репозиторий из методички и запускаем его.
![alt text](/screenshots/hw11-11.PNG?raw=true "Screenshot11")

2. Проверяем статус развёрнутых vm's, подключаемся к vm с именем  client, пробуем внести изменения в dns зону на сервере с ip 192.168.50.10 (vm ns01 ).
![alt text](/screenshots/hw11-12.PNG?raw=true "Screenshot12")

3. Так как изменения не получилось внести, смотрим на client логи и видим, что ошибок тут нет.
![alt text](/screenshots/hw11-13.PNG?raw=true "Screenshot13")

4. Подключаемся к vm c именем ns01 и проверяем там лог файлы и видим, что проблема в контексте безопасности. Смотрим контекст в директории `/etc/named`.
![alt text](/screenshots/hw11-14.PNG?raw=true "Screenshot14")

5. Смотрим, где должны лежать файлы dns сервера согласно политикам SELinux.
![alt text](/screenshots/hw11-15.PNG?raw=true "Screenshot15")

6. Меняем тип контекста на этом директории `/etc/named` и проверяем что он применился.
![alt text](/screenshots/hw11-16.PNG?raw=true "Screenshot16")

7. Возвращаемся на клиент и пытаемся повторить ранее вносимые изменения, проверяем что в этот раз они применилсь успешно.
![alt text](/screenshots/hw11-17.PNG?raw=true "Screenshot17")

8. Чтобы убедиться, что наши изменения контекста безопасности не слетят после перезагрузки, перезагружаем машины и делаем запрос с client снова.
![alt text](/screenshots/hw11-18.PNG?raw=true "Screenshot18")

9. Возвращаем контексты безопасности в первоначальное состояние.
![alt text](/screenshots/hw11-19.PNG?raw=true "Screenshot19")

### **Вариант 2 - Создание модуля**

1. На сервере с помощью утилиты audit2allow проверям лог и создаём с её помощью модуль для корректной работы named.service.
![alt text](/screenshots/hw11-20.PNG?raw=true "Screenshot20) 

2. Проверяем на client, помог ли наш модуль применённый на ns01.
![alt text](/screenshots/hw11-21.PNG?raw=true "Screenshot21)






